# Nugget Artifact Evaluation (AE)

This document describes how to set up the environment required to reproduce the steps in Nugget.

Throughout this document, we use:

- `[project dir]`: the root directory of the cloned Nugget repository.
- `papi install prefix`: the directory where PAPI is installed by the helper scripts.

## Reproducing Paper Experiments

For easy reproducing, we offer auto scripts for each step.
We expect the Docker image to be used for better environment control.
In our actual experiments, we ensured to minimize the noises in our measurement by using tools such as `cpuutils` and controlling the system such as fixing the CPU frequency.
Such things are hard to write an auto scripts to do because different systems might have different versions of the tools and might require different permissions.
Therefore, in the reproducing scripts, we do not enforce such operations to limit the noises in the measurements.

First, build the Docker image:

```bash
cd Docker
make image
make NUM_CORES_TO_USE=[the max number of cores you want the container to use] WORKDIR=$PWD/.. run
```

Because building LLVM can take have a big memory usage so setting the `NUM_CORES_TO_USE` to `$(nproc)` might not be ideal depending on the system you use.
You can read the `/Docker/Makefile` to see details of how the Docker image and container are built.

After getting into the Docker container with the project directory mounted as the workdir, run:

```bash
chmod +x install.sh
./install.sh
```

At the end, the script will output the environment variables that you should set.
You can set it up by copy, paste, and run in your terminal.

### Preparation and Interval Analysis

#### NPB

```bash
cd nugget-protocol-NPB
python3 ae-scripts/preparation_and_interval_analysis.py -d [project directory]
```
By default, it runs input class A with 4 threads.
It allows you to see the whole process quicker.
You can change the input class and number of threads by passing the options:

```bash
python3 ae-scripts/preparation_and_interval_analysis.py -d [project directory] -s [input class, i.e. A,B,C...] -t [the number of threads]
```

At the end of it, you will find the analysis results including the time it takes for each analysis under the `nugget-protocol-NPB/ae-experiments/[the binary of the input class]/[number of threads]`.

For example:
```terminal
dev@99494d3119ed:/workdir/nugget-protocol-NPB/ae-experiments/analysis$ ls
ir_bb_analysis_exe_bt_A  ir_bb_analysis_exe_ep_A  ir_bb_analysis_exe_is_A  ir_bb_analysis_exe_mg_A
ir_bb_analysis_exe_cg_A  ir_bb_analysis_exe_ft_A  ir_bb_analysis_exe_lu_A  ir_bb_analysis_exe_sp_A
dev@99494d3119ed:/workdir/nugget-protocol-NPB/ae-experiments/analysis$ ls ir_bb_analysis_exe_bt_A/
threads-4
dev@99494d3119ed:/workdir/nugget-protocol-NPB/ae-experiments/analysis$ ls ir_bb_analysis_exe_bt_A/threads-4/
execution_time.txt  stderr.log  stdout.log analysis-output.csv
```

The `analysis-output.csv` contains the LLVM IR BBV and the CSV information talked in the paper.
The `execution_time.txt` contains the time it takes for this analysis to finish.

### Sample Selection

Run k-means and optional random sampling to pick representative regions per benchmark and generate markers:

```bash
cd nugget-protocol-NPB
python3 ae-scripts/sample_selection.py -d [project dir] -s [input class] -b "CG EP" -t [threads] --k [num_clusters]
```

Example:

```bash
python3 ae-scripts/sample_selection.py -d [project dir]
```

The outputs are under the `ae-experiments` directory. 

### Nugget Creation and Sample Validation

Build nugget and naive binaries for the selected regions, run them, and emit measurement/prediction CSVs:

```bash
cd nugget-protocol-NPB
python3 ae-scripts/nugget_creation_and_validaton.py -d [project dir] -s [input class] -b "CG EP" -t [threads] --grace-perc 0.98
```

What happens:
- Configures and builds nugget and naive targets (using the selections/markers produced above).
- Runs naive binaries to record baselines, then runs each nugget binary (handles nested executable paths).
- Aggregates runtimes into `ae-experiments/nugget-measurement/measurements.csv`.
- Computes program-level predicted runtime using k-means cluster weights and writes prediction errors (k-means and random) to `ae-experiments/nugget-measurement/prediction-error.csv`.

Key options:
- `-s/--size`: input class (A/B/C...).
- `-b/--benchmarks`: space/comma/semicolon separated list, e.g., `"CG EP"`.
- `-t/--threads`: threads for the runs.
- `--grace-perc`: grace percentage used in marker generation (matches sample selection step).

### gem5 simulation

Please see under the gem5-experiment directory.

## Preparation

### 1. Dependencies and Tools

Make sure the following prerequisite is installed:

```bash
sudo apt install build-essential scons python3-dev git pre-commit zlib1g zlib1g-dev \
    libprotobuf-dev protobuf-compiler libprotoc-dev libgoogle-perftools-dev \
    libboost-all-dev  libhdf5-serial-dev python3-pydot python3-venv python3-tk mypy \
    m4 libcapstone-dev libpng-dev libelf-dev pkg-config wget cmake doxygen clang-format \
    libncurses-dev
```

Since Nugget measures using `perf`, so `perf` should be setup with correct permission in the system using for experiments.

We offer a Docker image that you can find in `/Docker` to have the above prerequisite installed.
We also has an `install.sh` that helps to install the LLVM compiler and the PAPI tool.

#### 1.1 LLVM with Nugget passes

This builds LLVM with the Nugget analysis and transformation passes.

```bash
cd [project dir]/llvm-project
./build_cmd.sh [llvm install prefix]
```

* Replace `[llvm install prefix]` with the directory where you want LLVM to be installed.
* After this step, your Nugget-enabled `clang`/`opt` etc. will live under that install prefix.

##### 1.1.1 Find the supported features 

Different machine supports different features with LLVM backend, running this script helps to find out automatically.

```bash
cd [project dir]/nugget_util/cmake/check-cpu-features
LLVM_BIN=[llvm install prefix]/bin LLVM_LIB=[llvm install prefix]/lib LLVM_INCLUDE=[llvm install prefix]/include make
./check-cpu-features
```

The reason why we separated `LLVM_BIN`, `LLVM_LIB`, and `LLVM_INCLUDE` is when using package LLVM, these three can be at different directories.
For example, `/usr/bin`, `/usr/lib/llvm-19/lib`, and `/usr/include/llvm-19/llvm`.

This is one example output you will find in `llc-command.txt` after running the above commands:

```bash
$ cat llc-command.txt
-mcpu=neoverse-n1 -mtriple=aarch64-unknown-linux-gnu -mattr="+fp-armv8,+lse,+neon,+crc,+crypto"
```

You can feed this into `llc` for backend optimization.

#### 1.2 PAPI (hardware performance counters)

PAPI is used to collect hardware performance counters for the Nugget runs.

```bash
cd [project dir]/nugget_util/hook_helper/other_tools/papi
./get-papi.sh
```

The tool will be installed under:

```text
[project dir]/nugget_util/hook_helper/other_tools/papi/[your system's arch]
```

For example:

```text
[project dir]/nugget_util/hook_helper/other_tools/papi/aarch64
```

We refer to this directory as `[papi install prefix]` in the rest of the document.

#### 1.3 Test and generate event combinations

This step:

* Tests the PAPI installation.
* Computes combinations of events so that you can cover all supported events in a minimal number of runs, given a fixed number of hardware counters.

We recommend setting `[# of events per run]` to the number of hardware counters on your machine.

You can get that information by running:

```bash
./[papi install prefix]/bin/papi_avail -a
```

Then run:

```bash
./install-test-papi-combos.sh [papi install prefix]
./test_papi_combos [# of events per run] [papi install prefix]/bin/papi_avail
```

> Note: `install-tset-papi-combos.sh` must be run from
> `[project dir]/nugget_util/hook_helper/other_tools/papi`
> so that relative paths work correctly.

If it complains about a missing `libpfm.so.4`, install the required library:

```bash
cd [project dir]/nugget_util/hook_helper/other_tools/papi/needed-lib
./install-pfm.sh
export LD_LIBRARY_PATH=$PWD/libpfm/lib:$LD_LIBRARY_PATH
```

You need to keep `LD_LIBRARY_PATH` set in any shell where you use this PAPI build.

##### Example output

Example:

```bash
./test_papi_combos 6 $PWD/aarch64/bin/papi_avail
```

Possible output (truncated):

```text
...
[SUPPORTED] ['PAPI_L1_DCR', 'PAPI_L1_DCW', 'PAPI_L2_DCW', 'PAPI_L1_ICH', 'PAPI_L1_ICA', 'PAPI_L2_TCA'],
[SUPPORTED] ['PAPI_L2_DCR', 'PAPI_L1_DCW', 'PAPI_L2_DCW', 'PAPI_L1_ICH', 'PAPI_L1_ICA', 'PAPI_L2_TCA'],

Tested 376740 combos, 129171 supported (34.3%)

Found a cover of size 5 (theoretical min 5):

[['PAPI_L1_DCM', 'PAPI_L1_ICM', 'PAPI_L2_DCM', 'PAPI_TLB_DM', 'PAPI_L2_LDM', 'PAPI_BR_MSP'],
 ['PAPI_STL_ICY', 'PAPI_HW_INT', 'PAPI_BR_PRC', 'PAPI_BR_INS', 'PAPI_RES_STL', 'PAPI_TOT_CYC'],
 ['PAPI_TOT_INS', 'PAPI_FP_INS', 'PAPI_LD_INS', 'PAPI_SR_INS', 'PAPI_VEC_INS', 'PAPI_LST_INS'],
 ['PAPI_L1_DCA', 'PAPI_L2_DCA', 'PAPI_L1_DCR', 'PAPI_L2_DCR', 'PAPI_L1_DCW', 'PAPI_L2_DCW'],
 ['PAPI_L1_ICM', 'PAPI_TOT_CYC', 'PAPI_SYC_INS', 'PAPI_L1_ICH', 'PAPI_L1_ICA', 'PAPI_L2_TCA']]
```

**What this program does**

`test_papi_combos` takes:

* `[# of events per run]`: the number of events per run (typically equal to the number of hardware counters).
* `papi_avail`: the PAPI availability binary.

It then:

1. Enumerates the possible event combinations of size `[# of events per run]`.
2. Finds which combinations are supported by the hardware.
3. Computes a minimal set of combinations that jointly cover all supported events.

In the example above:

* You need **5 runs** to cover all events.
* Each run can measure **6 events**.
* These combinations will be used later when collecting measurements.

Note: For multi-threaded programs, we can reliably measure only the runtime; detailed per-event measurements are mainly meaningful for single-threaded executions.

#### 1.4 gem5 m5ops

This is required when running Nugget under gem5 (for invoking gem5-specific hooks via `m5` ops).

```bash
cd [project dir]/nugget_util/hook_helper/other_tools/gem5
ISAS="[the abis you want to install]" ./get-gem5-util.sh
```

More information about which ABIs m5ops supports can be found in the
[gem5 documentation](https://github.com/gem5/gem5/blob/stable/util/m5/README.md#supported-abis).

The script will ask if you want to use a cross compiler for each ABI. Example:

```bash
ISAS="arm64 riscv" ./get-gem5-util.sh

...
Building for ISA: arm64
Enter CROSS_COMPILE for arm64 (leave blank for none): /usr/bin/aarch64-linux-gnu-
...
scons: done building targets.
Building for ISA: riscv
Enter CROSS_COMPILE for riscv (leave blank for none): /usr/bin/riscv64-linux-gnu-
...
```

After the script finishes, the built m5ops libraries/binaries will be placed in per-ABI subdirectories:

```text
$ ls
arm64  get-gem5-util.sh  include  README.md  riscv
```

#### 1.5 Sniper sim_api

This builds the Sniper `sim_api` used by Nugget when running on the Sniper simulator.

```bash
cd [project dir]/nugget_util/hook_helper/other_tools/sniper
make
```

### 2. LLVM IR files


