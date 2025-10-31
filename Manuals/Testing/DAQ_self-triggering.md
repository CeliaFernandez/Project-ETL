# Detailed Explanation of `self_trigger_daq.py`

## Overview

`self_trigger_daq.py` implements a **self-triggered data acquisition** mode where the ETROC chips themselves generate triggers when particles hit them, rather than relying on an external oscilloscope trigger. This is fundamentally different from `scope_daq.py`.

## Step-by-Step Execution Flow

### **Phase 1: Initialization (Lines 1-44)**

```python
# Lines 40-43
tb_config = load_config()
tb_config, reuse_thresholds_run_log = terminal_ui.init_config(tb_config)
```

**What happens:**
1. Loads TOML configuration from `configs/active_config/`
2. **Interactive prompts** appear asking you to:
   - Set number of runs, beam energy, comments
   - For each ETROC: offset, power mode, whether to reuse thresholds
3. If reusing thresholds, asks for path to previous run log file

**Output:** Displays ASCII art logo and telescope configuration table

---

### **Phase 2: Run Number Management (Lines 49-54)**

```python
run_number_path = daq_dir/Path('static/next_run_number.txt')
run_start = daq_utils.get_run_number(run_number_path)
num_runs = tb_config.run_config.num_runs
run_stop = run_start + num_runs - 1
```

**What happens:**
- Reads current run number from `daq/static/next_run_number.txt`
- Calculates run range (e.g., if next run is 1000 and you want 5 runs → 1000-1004)

---

### **Phase 3: Run Log Creation (Lines 56-64)**

```python
run_log_path = run_log_dir / Path(f"runs_{run_start}_{run_stop}.json")
run_log = {
    "config": tb_config.model_dump(mode="json"),
    "runs": [],
    "thresholds": {}
}
```

**What happens:**
- Creates JSON file to log all runs (e.g., `runs_1000_1004.json`)
- Stores full configuration snapshot
- Initializes empty arrays for run data and thresholds

---

### **Phase 4: ETROC Configuration (Lines 67-86)**

```python
etl_telescope = ETL_Telescope(telescope_config)
etl_telescope.configure_etrocs(reuse_thresholds_run_log=reuse_thresholds_run_log)
etl_telescope.plot_etroc_thresholds(threshold_dir=run_log_dir)
```

#### **What happens in `ETL_Telescope.__init__`** (`etl_telescope.py:74-116`):
1. Connects to KCU105 FPGA via IPbus protocol
2. Tests communication with loopback test
3. For each readout board (RB):
   - Connects to VTRX+ optical transceiver
   - Reads multiplexer and LPGBT ADC values
   - Connects to modules via hard reset
   - Tests DAQ by sending 10 test L1A triggers
4. **Prompts: "Turn on HV"** - waits for you to enable high voltage

#### **What happens in `configure_etrocs`** (`etl_telescope.py:202-246`):
For each ETROC chip:
1. **Sets power mode** (low/medium/high affects timing performance)
2. **Sets L1A delay** - timing offset between trigger and readout
3. **Either:**
   - **Runs threshold scan** (`etroc.run_threshold_scan()`) - measures noise in each pixel
     - Returns 16×16 baseline and noise_width arrays
     - Sets DAC threshold = baseline + offset for each pixel
   - **OR reuses thresholds** from previous run log
4. **Enables self-trigger mode** (`etroc.configure_self_trig()`) - line 243
5. **Enables data readout**

#### **What happens in `plot_etroc_thresholds`** (`etl_telescope.py:296-319`):
- Saves heatmap images of baseline and noise_width for each ETROC
- Saves histograms showing distribution statistics

**Saves to run log:**
```python
run_log["thresholds"] = etl_telescope.thresholds
```

---

### **Phase 5: DAQ Stream Setup (Line 90)**

```python
with daq_utils.RunDaqStreamPY(tb_config, daq_dir) as daq_stream:
```

**What happens in `RunDaqStreamPY.__init__`** (`daq_utils.py:94-115`):
- Sets up paths to status files:
  - `start_acquiring.txt` - lock file controlling when to acquire
  - `is_rb_{rb}_ready.txt` - status files for each readout board
- Sets `start_acquiring` flag to False on entry

**Key mechanism:** This class will launch `daq_stream.py` as a subprocess

---

### **Phase 6: Beam Confirmation (Lines 91-93)**

```python
user_input_for_beam_on = input("Is the beam on? (y/abort) ")
while not daq_utils.is_beam_on(user_input_for_beam_on):
    user_input_for_beam_on = input("Need somebody to turn the beam on! Is it on? (y/abort) ")
```

**What happens:**
- Waits for operator to confirm particle beam is active
- Typing "abort" exits the program
- Only "y" proceeds

---

### **Phase 7: Data Acquisition Loop (Lines 96-125)**

This is the **core DAQ logic**:

```python
for run in range(run_start, run_start+num_runs):
```

#### **Step 7a: Launch DAQ Stream Subprocess (Lines 99-101)**

```python
daq_stream.execute_python_subprocess()
while not daq_stream.rbs_are_ready:
    time.sleep(0.5)
```

**What happens:**
1. **Spawns subprocess** running `daq_stream.py` with arguments:
   ```bash
   python daq_stream.py \
     --l1a_rate 0 \
     --kcu 192.168.0.10 \
     --rb 0 \
     --run 1000 \
     --lock /path/to/start_acquiring.txt \
     --binary_dir /path/to/binaries \
     --daq_static_dir /path/to/static
   ```

2. **Inside `daq_stream.py`** (`daq_stream.py:74-256`):
   - Connects to KCU via uhal
   - Resets FIFO buffers on readout board
   - Sets L1A trigger rate to 0 (we're self-triggering!)
   - **Waits in lock loop** (lines 115-121):
     ```python
     while (Running.lower() == "false" or Running.lower() == "stop"):
         if iteration == 0:
             write_rb_is_ready(f"/is_rb_{rb}_ready.txt")  # Signals ready!
         Running = get_kcu_flag(lock_path=lock)  # Checks start_acquiring.txt
     ```
   - Continuously checks `start_acquiring.txt` until it says "True"

3. **Back in main script:** Polls `is_rb_{rb}_ready.txt` files until all are "True"

#### **Step 7b: Start Self-Triggered Acquisition (Lines 103-107)**

```python
daq_stream.start_acquiring = True  # Sets start_acquiring.txt to "True"
etl_telescope.self_trig_rb.self_trig_start()  # Enables self-trigger on RB
logging.info(f"Started acquisition {DAQ_RUN_TIME} seconds")
time.sleep(DAQ_RUN_TIME)  # Acquires for 30 seconds!
```

**What happens:**

1. **`start_acquiring = True`** writes "True" to `start_acquiring.txt`
   - This releases the waiting `daq_stream.py` subprocess

2. **`self_trig_start()`** (in `module_test_sw/tamalero/ReadoutBoard.py`):
   - Writes to KCU register: `READOUT_BOARD_{rb}.SELF_TRIG_EN = 1`
   - **ETROCs now generate triggers when particles hit!**

3. **Inside `daq_stream.py` (lines 125-140):**
   ```python
   while (Running.lower() != "false"):
       occupancy = get_occupancy(hw, rb)  # Check FIFO buffer
       num_blocks_to_read = occupancy // 128

       if num_blocks_to_read > 0:
           for x in range(num_blocks_to_read):
               reads += [hw.getNode(f"DAQ_RB{rb}").readBlock(128)]
               hw.dispatch()  # Read data from FIFO into memory
   ```
   - **Continuously polls** the FIFO buffer on the KCU
   - When occupancy ≥ 128 words, reads a block
   - Stores data in RAM (not written to disk yet)

4. **Main script sleeps for 30 seconds** while data flows

#### **Step 7c: Stop Acquisition (Lines 108-111)**

```python
daq_stream.start_acquiring = False  # Sets start_acquiring.txt to "False"
daq_stream.wait_til_done()
etl_telescope.self_trig_rb.self_trig_stop()
```

**What happens:**

1. **`start_acquiring = False`** breaks the loop in `daq_stream.py`

2. **`daq_stream.py` finishes** (lines 162-232):
   - Sets L1A rate back to 0
   - **Reads remaining data from FIFO:**
     ```python
     occupancy = get_occupancy(hw, rb)
     num_blocks_to_read = occupancy // block
     remainder = occupancy % block
     # Reads all remaining blocks + remainder
     ```
   - **Writes binary file:** `output_run_{run}_rb{rb}.dat`
     - Contains packed 32-bit words with ETROC hit data
   - **Writes YAML log:** `log_run_{run}_rb{rb}.yaml`
     - Contains: L1A rate, number of events, lost events, speed (Mbps)
   - Appends run number to `daq_log.txt`

3. **`wait_til_done()`** blocks until subprocess exits

4. **`self_trig_stop()`** disables self-trigger on RB

#### **Step 7d: Update Run Log (Lines 113-120)**

```python
daq_utils.run_log_update(run, tb_config, etl_telescope, start_time)
```

**What happens in `run_log_update`** (`daq_utils.py:55-92`):
1. Reads HV supply log files to get:
   - Bias voltage (VMon)
   - Leakage current (IMon)
2. Reads ETROC chip temperatures via ADC
3. **Appends to run log JSON:**
   ```json
   {
     "run_number": 1000,
     "start_time": "2025-10-31T14:23:10",
     "finish_time": "2025-10-31T14:23:45",
     "module_209_bias_voltage": 185.3,
     "module_209_leakage_current": 12.4,
     "module_209_chip_0_temperature": 0.847
   }
   ```

#### **Step 7e: Increment Run Number (Line 124)**

```python
daq_utils.set_run_number(run_number_path, run=run+1)
```

**Loop repeats** for next run!

---

## Key Differences from `scope_daq.py`

| Feature | `scope_daq.py` | `self_trigger_daq.py` |
|---------|----------------|----------------------|
| **Trigger source** | LeCroy oscilloscope (MCP signal) | ETROC self-trigger (particle hits) |
| **Acquisition end** | Fixed event count (scope triggers) | Fixed time (30 seconds) |
| **Oscilloscope** | Required, saves waveforms | Not used |
| **L1A rate** | 0 (external trigger) | 0 (self trigger) |
| **Watchdog merging** | Waits for scope + ETROC binaries | Only waits for ETROC binaries |

---

## Data Files Created Per Run

For run 1000 with 1 readout board:

```
/data/etroc_binaries/
├── output_run_1000_rb0.dat          # Binary ETROC hit data
├── log_run_1000_rb0.yaml            # DAQ statistics
└── /run_logs/
    ├── runs_1000_1004.json          # Metadata for all runs
    ├── baseline_rb_0_module_209_etroc_0.png
    ├── noise_width_rb_0_module_209_etroc_0.png
    ├── baseline_rb_0_module_209_etroc_0_histogram.png
    └── noise_width_rb_0_module_209_etroc_0_histogram.png
```

---

## What Happens in Background (Watchdog)

If `daq_watchdog.py` is running (`processing/daq_watchdog.py`):

1. Monitors `etroc_binary_data_directory`
2. When `output_run_1000_rb0.dat` appears
3. **Merges to ROOT format** using `uproot`
4. **Moves files** to archive directory

---

## Critical Hardware Communication

```
self_trigger_daq.py
    ↓
ETL_Telescope (Python)
    ↓
module_test_sw.tamalero (Python wrapper)
    ↓
uhal (IPbus protocol)
    ↓
Ethernet → KCU105 FPGA
    ↓
Optical fibers (VTRX+)
    ↓
Readout Board (RB)
    ↓
ETROC chips (16×16 pixels each)
```

---

## Self-Trigger Mechanism

When `self_trig_start()` is called:

1. **ETROC discriminators** compare pixel signals to thresholds
2. **When signal > threshold:** ETROC records:
   - Time of arrival (TOA) - 10 bits
   - Time over threshold (TOT) - 9 bits
   - Pixel row/column
3. **ETROC sends trigger to RB** via trigger link
4. **RB forwards to KCU** which increments event counter
5. **KCU reads ETROC data** from circular buffer via I2C
6. **Data flows to FIFO** where `daq_stream.py` polls it

This all happens **autonomously** - no external trigger needed!

---

## How to Run

```bash
# 1. Source environment
source setup.sh

# 2. In one terminal, start the watchdog (optional but recommended)
cd processing
python daq_watchdog.py

# 3. In another terminal, run the DAQ
cd daq
python self_trigger_daq.py
```

### What You'll See:

1. ASCII art "TEST BEAM" logo
2. Telescope configuration visualization
3. Prompts for:
   - Number of runs
   - Beam energy
   - Comments
   - Per-ETROC settings (offset, power mode, reuse thresholds)
4. ETROC connection messages
5. "Turn on HV" prompt
6. Threshold scan progress (or "Reusing thresholds" messages)
7. "Is the beam on? (y/abort)" prompt
8. For each run:
   - "ACQUIRING RUN {run}" message
   - 30 second acquisition
   - "FINISHED RUN {run}" message
   - Temperature and HV readings

---

## Configuration Requirements

In your `configs/active_config/*.toml`, you need:

```toml
[run_config]
num_runs = 5
beam_energy = 120  # GeV

[telescope_config]
l1a_delay = 508  # Timing offset for readout

[[telescope_config.service_hybrids]]
rb = 0
self_triggering = true  # MUST be true for self_trigger_daq.py!

[[telescope_config.service_hybrids.modules]]
name = "Module_209"
id = 209
slot = 0
offset = {0 = 30, 1 = 30}  # ETROC 0 and 1 thresholds
power_mode = {0 = "high", 1 = "high"}
reuse_thresholds = {0 = false, 1 = false}
```

---

## Troubleshooting

### "No modules connected, aborting..."
- Check fiber connections to readout board
- Verify KCU IP address is correct and reachable
- Ensure `ipbus` controlhub is running: `/opt/cactus/bin/controlhub_start`

### "uhal UDP error in reading FIFO"
- Network congestion or packet loss
- Check ethernet cable and switch
- Verify KCU has dedicated ethernet port (not sharing bandwidth)

### "No communications with KCU105... quitting"
- IPbus not responding
- Check KCU power and network connection
- Ping the KCU IP address

### Threshold scan fails
- ETROC may not be properly connected
- Check I2C communication
- Verify module power is stable

### Self-trigger doesn't produce events
- Check discriminator thresholds aren't too high
- Verify beam is actually hitting the detector
- Check that `self_triggering = true` in config

---

## Important Constants

```python
DAQ_RUN_TIME = 30  # seconds per run (line 34)
```

To change acquisition time, edit this value in `self_trigger_daq.py:34`

---

## File References

| File | Key Functions | Purpose |
|------|---------------|---------|
| `daq/self_trigger_daq.py` | Main script | Orchestrates self-triggered DAQ |
| `daq/etl_telescope.py:74` | `ETL_Telescope.__init__` | Connects to KCU and readout boards |
| `daq/etl_telescope.py:202` | `configure_etrocs` | Sets up ETROC chips for self-trigger |
| `daq/etl_telescope.py:363` | `self_trig_rb` property | Gets RB configured for self-trigger |
| `daq/daq_stream.py:74` | `stream_daq` | Subprocess that reads FIFO data |
| `daq/daq_utils.py:94` | `RunDaqStreamPY` | Manages daq_stream subprocess |
| `daq/daq_utils.py:55` | `run_log_update` | Logs HV and temperature data |
| `daq/terminal_ui.py:110` | `init_config` | Interactive configuration prompts |
| `config.py` | Pydantic models | Type-safe configuration validation |

---

## Summary

`self_trigger_daq.py` enables **autonomous data collection** where the ETROC detector chips themselves decide when to record data based on particle hits, rather than relying on an external reference detector. This is useful for:

- **Commissioning** new modules without MCP
- **High-rate testing** where scope would be overwhelmed
- **Long-duration studies** with continuous triggering
- **Detector characterization** of self-trigger performance

The 30-second time-based acquisition allows you to collect statistics on self-trigger rates and characterize detector performance under various conditions.
