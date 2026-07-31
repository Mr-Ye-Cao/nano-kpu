# Nano-KPU Apple Silicon worklog

Last updated: 2026-07-30

## Goal

Configure this checkout to run the native RTL simulation and Yosys synthesis
flow on Ye's Apple Silicon Mac, then transfer the complete checkout to
`ye-ubuntu` and run the full evaluation there through completion.

Repository:
`/Users/yecao/Documents/loopcell-jul12-onwards/technical/synthetic-exploration/nano-kpu`

Host detected:

- Operating system: macOS (`Darwin`)
- Architecture: Apple Silicon (`arm64`)
- Conda: `/opt/homebrew/bin/conda`
- Homebrew prefix: `/opt/homebrew`
- Initial state: Verilator and Yosys were not installed

## Done

- Confirmed the checkout began clean on `main`.
- Read the repository setup, evaluation entry points, and parent repository
  instructions.
- Confirmed conda-forge publishes native Apple Silicon Verilator 5 builds but
  does not publish native Apple Silicon Yosys builds.
- Confirmed Homebrew provides Yosys 0.67, satisfying the repository's
  `yosys >= 0.64` requirement.
- Updated `scripts/setup_env.sh` to:
  - detect the operating system and CPU architecture;
  - choose the correct Miniconda bootstrap installer for macOS and Linux;
  - create a native Conda environment with Verilator 5, Python 3.12, and
    NumPy on Apple Silicon;
  - install or reuse Homebrew Yosys on Apple Silicon;
  - validate the external Yosys binary;
  - write both the Conda environment and Homebrew tool directory into the
    generated `scripts/kpu-env.sh` path setup.
- Kept the frozen `README.md` unchanged so the evaluation integrity check
  remains valid; Apple Silicon setup details live in this worklog and in the
  setup script comments/output.
- Verified that no nano-KPU process remained on the Mac before remote work.
- Restored the documented `ye-ubuntu` SSH route through `ye-claw` and the
  `ssh-machine-map` wake procedure. The wake script needed
  `/opt/homebrew/bin` added to its non-interactive `PATH` so it could find the
  already-installed `wakeonlan` command.
- Copied the complete 60 MiB checkout to:
  `/home/ye/Documents/loopcell-jul12-onwards/technical/synthetic-exploration/nano-kpu`
  on `ye-ubuntu`. The copy included `.git`, ignored build artifacts, the
  downloaded Liberty file, and this worklog. A checksum dry run found no
  transfer differences.
- Ran the Linux setup successfully on `ye-ubuntu` and regenerated
  `scripts/kpu-env.sh` with Ubuntu-native paths.
- Ran `python3 harness/evaluate.py` in a persistent remote `tmux` session and
  monitored it through successful completion with exit code 0.

## Doing

- No nano-KPU process is currently running on either the Mac or `ye-ubuntu`.
- The remote `tmux` session exited normally after writing the final report.

## Plan / optional next work

The Mac setup, remote transfer, and full Ubuntu evaluation are complete.
Optional follow-ups are:

1. Commit the setup script and this worklog if these local changes should be
   retained in repository history.
2. Pull selected ignored Ubuntu build artifacts back to the Mac only if a local
   copy is useful; the authoritative completed outputs currently remain on
   `ye-ubuntu`.

The full evaluation and synthesis commands are documented below, but they may
take hours and are not required to prove that the local toolchain starts and
the RTL functional path works.

## Exact reproduction commands

Run from a fresh shell:

```bash
cd /Users/yecao/Documents/loopcell-jul12-onwards/technical/synthetic-exploration/nano-kpu
bash scripts/setup_env.sh
source scripts/kpu-env.sh
verilator --version
yosys -V
python3 -c 'import sys, numpy; print(sys.version); print("numpy", numpy.__version__)'
make selftest
make lint
make audit
python3 harness/evaluate.py --quick
```

Run synthesis-only area and timing estimation:

```bash
cd /Users/yecao/Documents/loopcell-jul12-onwards/technical/synthetic-exploration/nano-kpu
source scripts/kpu-env.sh
make synth
```

Run the full evaluation, including functional simulation and synthesis:

```bash
cd /Users/yecao/Documents/loopcell-jul12-onwards/technical/synthetic-exploration/nano-kpu
source scripts/kpu-env.sh
python3 harness/evaluate.py
```

Run all functional checks while skipping synthesis:

```bash
cd /Users/yecao/Documents/loopcell-jul12-onwards/technical/synthetic-exploration/nano-kpu
source scripts/kpu-env.sh
python3 harness/evaluate.py --skip-synth
```

## Exact ye-ubuntu transfer and run commands

If `ye-ubuntu` is offline in Tailscale, wake it through `ye-claw`:

```bash
ssh ye-claw 'export PATH=/opt/homebrew/bin:$PATH; /Users/ye-claw/ye-ubuntu/wake-ye-ubuntu.sh --unlock'
ssh -o BatchMode=yes -o ConnectTimeout=8 ye-ubuntu 'hostname; whoami; uptime'
```

Copy the complete checkout from the Mac without deleting either side:

```bash
cd /Users/yecao/Documents/loopcell-jul12-onwards/technical/synthetic-exploration/nano-kpu
remote_parent=/home/ye/Documents/loopcell-jul12-onwards/technical/synthetic-exploration
ssh ye-ubuntu "mkdir -p '$remote_parent'"
rsync -az --progress -e 'ssh -o BatchMode=yes -o ServerAliveInterval=30' ./ ye-ubuntu:"$remote_parent/nano-kpu/"
rsync -aznc -e 'ssh -o BatchMode=yes' ./ ye-ubuntu:"$remote_parent/nano-kpu/"
```

Configure the native Linux environment:

```bash
ssh ye-ubuntu 'cd /home/ye/Documents/loopcell-jul12-onwards/technical/synthetic-exploration/nano-kpu && bash scripts/setup_env.sh'
```

Launch the full evaluation persistently with an exit marker:

```bash
ssh ye-ubuntu 'repo=/home/ye/Documents/loopcell-jul12-onwards/technical/synthetic-exploration/nano-kpu; cd "$repo"; tmux new-session -d -s nano-kpu-full "cd $repo && source scripts/kpu-env.sh && python3 harness/evaluate.py > build/ubuntu-full-evaluate.log 2>&1; rc=\$?; echo \$rc > build/ubuntu-full-evaluate.exit"'
```

Monitor and inspect the result:

```bash
ssh ye-ubuntu 'cd /home/ye/Documents/loopcell-jul12-onwards/technical/synthetic-exploration/nano-kpu && tail -f build/ubuntu-full-evaluate.log'
ssh ye-ubuntu 'cd /home/ye/Documents/loopcell-jul12-onwards/technical/synthetic-exploration/nano-kpu && cat build/ubuntu-full-evaluate.exit && cat build/score.json'
```

## Setup logs

If setup fails, inspect the relevant log:

```bash
tail -200 /tmp/nano-kpu-conda-create.log
tail -200 /tmp/nano-kpu-brew-yosys.log
```

The setup-generated `scripts/kpu-env.sh`, downloaded Liberty file, and build
outputs are intentionally ignored by Git.

## Verification results

- Shell syntax: passed (`bash -n scripts/setup_env.sh`).
- Patch whitespace: passed (`git diff --check`).
- Toolchain setup: passed.
  - Verilator: 5.050
  - Yosys: 0.67+post
  - Python: 3.12 in the dedicated environment
  - NumPy: 2.5.1
  - Nangate45 Liberty SHA-256: matched the pinned digest
- Reference self-test and harness self-check: passed.
- ROM integrity: all 15 files passed.
- Verilator RTL lint: passed with zero errors and zero warnings.
- RTL audit: hard checks passed. It reported only the existing soft notices
  about simulation `$display` statements.
- First quick functional simulation: computation passed with `cos_min=1.0`,
  argmax agreement `1.0`, and valid protocol/weight-stream checks. It processed
  16 tokens in 177,591 cycles. The command exited nonzero only because the
  temporary README documentation edit triggered the frozen-file integrity
  check; that edit has been removed before the clean rerun.
- Clean quick functional simulation: passed with exit code 0.
  - Integrity: passed
  - Audit: passed
  - Lint: zero errors and zero warnings
  - Correctness: `cos_min=1.0`, argmax agreement `1.0`
  - Protocol and weight-stream checks: passed
  - Workload: 16 tokens in 177,591 cycles
  - Debug throughput: 11,099.438 cycles/token and 9,009.4656 tokens/s at the
    assumed 100 MHz target clock; quick mode skips timing synthesis
- Setup idempotency rerun: passed. It reused the Conda environment and Yosys,
  regenerated `scripts/kpu-env.sh`, and verified the existing Liberty digest
  without reinstalling or redownloading anything.
- Synthesis attempt: intentionally interrupted at the user's request.
  - `make synth` successfully invoked Homebrew Yosys on the complete RTL.
  - RTL elaboration and ABC mapping ran successfully.
  - The run entered the CPU-intensive final mapped-netlist analysis and was
    stopped with `Ctrl-C` before final area/timing JSON was produced.
  - No synthesis process remains active.
- Complete `ye-ubuntu` evaluation: passed with exit code 0.
  - Integrity, audit, lint, correctness, protocol, and weight-stream checks:
    passed
  - Long workload: 48 tokens in 550,420 cycles, `cos_min=1.0`, argmax
    agreement `1.0`
  - Short workload: 16 tokens in 177,591 cycles, `cos_min=1.0`, argmax
    agreement `1.0`
  - Randomized-reset comparison: matched
  - Critical path: 7.71125 ns; meets the 10 ns/100 MHz target
  - Logic area: 1.788648 mm2
  - Macro area: 2.181137 mm2
  - Total area: 3.969785 mm2 against the 4.0 mm2 budget
  - SRAM macros: 2,287,088 bits / 0.272642 MiB
  - Throughput: 11,467.083 cycles/token and 8,720.61335 tokens/s at 100 MHz
  - Final score: `/home/ye/Documents/loopcell-jul12-onwards/technical/synthetic-exploration/nano-kpu/build/score.json`
  - Full log: `/home/ye/Documents/loopcell-jul12-onwards/technical/synthetic-exploration/nano-kpu/build/ubuntu-full-evaluate.log`

## Current tracked changes

- `scripts/setup_env.sh`: modified for native Apple Silicon setup.
- `WORKLOG.md`: added for status, results, and reproduction instructions.
- `README.md`: unchanged, preserving the repository integrity manifest.
