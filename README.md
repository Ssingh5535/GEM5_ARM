
# gem5 ARM SE: Minimal Build, m5-annotated Workload, and Two-Level Cache Demo

> **Goal**: I built and ran a minimal **ARM (AArch64)** system in gem5 (SE mode), then extended it to a **two-level cache** hierarchy and compared statistics. This README captures my exact steps, commands, scripts, and a few troubleshooting notes. 

---

## 0) Environment

- Distro: Ubuntu 24.04
- gem5: `v23.0.0.1` (I built from source)
- Python: 3.12 (system)
- Toolchain: `g++-aarch64-linux-gnu`
- I worked from the gem5 repo root, e.g.:

```bash
cd ~/sim/gem5-x86/gem5
```

> I used **SE (syscall emulation) mode** in this tutorial.

---

## 1) Build gem5 for ARM (SE)

I built the ARM binary (`build/ARM/gem5.opt`):

```bash
scons -j"$(nproc)" build/ARM/gem5.opt
```

If I want a multi-ISA binary instead, I can do `build/ALL/gem5.opt`, but this walkthrough uses the **ARM** build.

---

## 2) Build `m5` tool + library for ARM64

I built the `m5` helper and static library, which I link into my AArch64 test program to bracket a region-of-interest (ROI) with `m5_reset_stats`, `m5_dump_stats`, and end the sim with `m5_exit`:

```bash
cd util/m5
scons -j"$(nproc)" build/arm64/out/m5 build/arm64/out/libm5.a
cd -
```

I verified the symbols exist (optional):

```bash
nm -g util/m5/build/arm64/out/libm5.a | egrep ' m5_(reset_stats|dump_stats|exit)$'
```

---

## 3) Build a tiny **AArch64** program with `m5` ops

I created `configs/tutorial/part1/arm_hello_m5.c`:

```c
#include <stdio.h>
#include <gem5/m5ops.h>

int main() {
    m5_reset_stats(0, 0);
    printf("Hello from ARM with m5!\\n");
    m5_dump_stats(0, 0);
    m5_exit(0);
    return 0;
}
```

I compiled **for AArch64** and linked against the `m5` static library. The `-mbranch-protection=none` avoids “bti unimplemented” warnings in some gem5 builds.

> **Static** is ideal, but if my distro lacks static glibc, I can drop `-static`.

```bash
# dynamic (widely compatible):
aarch64-linux-gnu-g++ -O2 -mbranch-protection=none \
  -I"$PWD/include" \
  -L"$PWD/util/m5/build/arm64/out" -lm5 \
  -o configs/tutorial/part1/arm-hello-m5 configs/tutorial/part1/arm_hello_m5.c

# static (preferred if available):
# aarch64-linux-gnu-g++ -O2 -static -mbranch-protection=none \
#   -I"$PWD/include" \
#   -L"$PWD/util/m5/build/arm64/out" -lm5 \
#   -o configs/tutorial/part1/arm-hello-m5 configs/tutorial/part1/arm_hello_m5.c
```

Sanity check:

```bash
file configs/tutorial/part1/arm-hello-m5   # ELF 64-bit ... ARM aarch64
chmod +x configs/tutorial/part1/arm-hello-m5
```

---

## 4) **Classic** simple config (ARM, no caches)

I wrote a classic tutorial-style config `configs/tutorial/part1/simple_arm.py` (no caches: CPU → membus → DDR3).

```python
# configs/tutorial/part1/simple_arm.py
import m5
from pathlib import Path
from m5.objects import (
    System, SrcClockDomain, VoltageDomain, AddrRange,
    SystemXBar, TimingSimpleCPU,
    MemCtrl, DDR3_1600_8x8,
    Process, SEWorkload, Root
)

system = System()

system.clk_domain = SrcClockDomain()
system.clk_domain.clock = "8GHz"
system.clk_domain.voltage_domain = VoltageDomain()

system.mem_mode = "timing"
system.mem_ranges = [AddrRange("8GiB")]

system.cpu = TimingSimpleCPU()
system.cpu.createInterruptController()   # required

system.membus = SystemXBar()

# No caches: connect CPU I/D directly to membus
system.cpu.icache_port = system.membus.cpu_side_ports
system.cpu.dcache_port = system.membus.cpu_side_ports

system.system_port = system.membus.cpu_side_ports

system.mem_ctrl = MemCtrl()
system.mem_ctrl.dram = DDR3_1600_8x8()
system.mem_ctrl.dram.range = system.mem_ranges[0]
system.mem_ctrl.port = system.membus.mem_side_ports

here = Path(__file__).resolve().parent
binary = str((here / "arm-hello-m5").resolve())

system.workload = SEWorkload.init_compatible(binary)
process = Process()
process.cmd = [binary]
system.cpu.workload = process
system.cpu.createThreads()

root = Root(full_system=False, system=system)
m5.instantiate()

print("Beginning simulation!")
exit_event = m5.simulate()
print("Exiting @ tick {} because {}".format(m5.curTick(), exit_event.getCause()))
```

Run it (write to an explicit `outdir` so my artifacts don’t land in `m5out/`):

```bash
./build/ARM/gem5.opt configs/tutorial/part1/simple_arm.py --outdir="$(pwd)/out/arm_nocache"
```

**Screenshot placeholder**: _ARM simple run output_  
`![ARM Simple Run](Images/arm_simple_run.png)`

**Screenshot placeholder**: _Outdir listing (no cache)_  
`![Outdir arm_nocache](Images/arm_nocache_ls.png)`

If I forget `--outdir`, gem5 writes to `m5out/` by default.

---

## 5) Extract ROI stats (between m5_reset_stats and m5_dump_stats)

I pulled the ROI block (2nd “Begin/End Simulation Statistics”) and printed a few lines:

```bash
awk '/Begin Simulation Statistics/{b++} b==2{print} /End Simulation Statistics/{if(b==2)exit}' \
  out/arm_nocache/stats.txt > out/arm_nocache/roi_stats.txt || true
[ -s out/arm_nocache/roi_stats.txt ] || cp out/arm_nocache/stats.txt out/arm_nocache/roi_stats.txt

echo "== core metrics (no cache, ROI) =="
grep -E '^(sim_seconds|sim_ticks|sim_freq|host_inst_rate|host_op_rate)\\b' out/arm_nocache/roi_stats.txt
grep -E '^(system\\.cpu\\.)?(committedInsts|numCycles|ipc)\\b' out/arm_nocache/roi_stats.txt || true
```

**Screenshot placeholder**: _ROI stats no-cache_  
`![ROI No Cache](Images/roi_nocache.png)`

---

## 6) Add L1I/L1D + shared L2 via a small helper (`caches.py`)

I created `configs/tutorial/part1/caches.py` (parameters compatible with v23):

```python
# configs/tutorial/part1/caches.py
from m5.objects import Cache

class L1Cache(Cache):
    assoc = 2
    tag_latency = 1
    data_latency = 1
    response_latency = 1
    mshrs = 16
    tgts_per_mshr = 20

    def connectCPU(self, cpu):
        raise NotImplementedError

    def connectBus(self, bus):
        # L1 requests go "down" toward L2 via cpu_side_ports
        self.mem_side = bus.cpu_side_ports

class L1ICache(L1Cache):
    size = '16kB'
    def connectCPU(self, cpu):
        self.cpu_side = cpu.icache_port

class L1DCache(L1Cache):
    size = '64kB'
    def connectCPU(self, cpu):
        self.cpu_side = cpu.dcache_port

class L2Cache(Cache):
    size = '256kB'
    assoc = 8
    tag_latency = 10
    data_latency = 10
    response_latency = 10
    mshrs = 20
    tgts_per_mshr = 12

    def connectCPUSideBus(self, bus):
        # L2 responds "upward" to L1s via mem_side_ports
        self.cpu_side = bus.mem_side_ports

    def connectMemSideBus(self, bus):
        # L2 requests "downward" to memory via cpu_side_ports
        self.mem_side = bus.cpu_side_ports
```

Then I copied my simple config to `two_level.py` and wired the caches and an L2 crossbar:

```python
# configs/tutorial/part1/two_level.py
import m5
from pathlib import Path
from m5.objects import (
    System, SrcClockDomain, VoltageDomain, AddrRange,
    SystemXBar, L2XBar, TimingSimpleCPU,
    MemCtrl, DDR3_1600_8x8,
    Process, SEWorkload, Root
)
from caches import L1ICache, L1DCache, L2Cache

system = System()
system.clk_domain = SrcClockDomain()
system.clk_domain.clock = "8GHz"
system.clk_domain.voltage_domain = VoltageDomain()

system.mem_mode = "timing"
system.mem_ranges = [AddrRange("8GiB")]

system.cpu = TimingSimpleCPU()
system.cpu.createInterruptController()

system.membus = SystemXBar()

# L1s
system.cpu.icache = L1ICache()
system.cpu.dcache = L1DCache()
system.cpu.icache.connectCPU(system.cpu)
system.cpu.dcache.connectCPU(system.cpu)

# L2 crossbar
system.l2bus = L2XBar()
system.cpu.icache.connectBus(system.l2bus)
system.cpu.dcache.connectBus(system.l2bus)

# Shared L2
system.l2cache = L2Cache()
system.l2cache.connectCPUSideBus(system.l2bus)
system.l2cache.connectMemSideBus(system.membus)

system.system_port = system.membus.cpu_side_ports

system.mem_ctrl = MemCtrl()
system.mem_ctrl.dram = DDR3_1600_8x8()
system.mem_ctrl.dram.range = system.mem_ranges[0]
system.mem_ctrl.port = system.membus.mem_side_ports

here = Path(__file__).resolve().parent
binary = str((here / "arm-hello-m5").resolve())

system.workload = SEWorkload.init_compatible(binary)
process = Process()
process.cmd = [binary]
system.cpu.workload = process
system.cpu.createThreads()

root = Root(full_system=False, system=system)
m5.instantiate()

print("Beginning simulation (two-level caches)!")
exit_event = m5.simulate()
print("Exiting @ tick {} because {}".format(m5.curTick(), exit_event.getCause()))
```

I ran it:

```bash
./build/ARM/gem5.opt configs/tutorial/part1/two_level.py --outdir="$(pwd)/out/arm_l1l2"
```

**Screenshot placeholder**: _ARM two-level run output_  
`![ARM Two-Level Run](Images/arm_l1l2_run.png)`

---

## 7) Extract cache stats (ROI) and compare

I pulled the ROI block and grabbed key counters:

```bash
awk '/Begin Simulation Statistics/{b++} b==2{print} /End Simulation Statistics/{if(b==2)exit}' \
  out/arm_l1l2/stats.txt > out/arm_l1l2/roi_stats.txt || true
[ -s out/arm_l1l2/roi_stats.txt ] || cp out/arm_l1l2/stats.txt out/arm_l1l2/roi_stats.txt

echo "== core metrics (L1+L2, ROI) =="
grep -E '^(sim_seconds|sim_ticks|sim_freq|host_inst_rate|host_op_rate)\\b' out/arm_l1l2/roi_stats.txt
grep -E '^(system\\.cpu\\.)?(committedInsts|numCycles|ipc)\\b' out/arm_l1l2/roi_stats.txt || true

echo
echo "== caches (ROI) =="
grep -Ei '(icache|dcache|l2).*(overall_(hits|misses|miss_rate))' out/arm_l1l2/roi_stats.txt || true
grep -Ei '(icache|dcache|l2).*((overallMissRate)|(overallHits)|(overallMisses))' out/arm_l1l2/roi_stats.txt || true
```

For my “hello + m5” workload I observed (representative excerpt):

```
system.cpu.dcache.overallHits::total      211
system.cpu.dcache.overallMisses::total     23
system.cpu.dcache.overallMissRate::total 0.098291

system.cpu.icache.overallHits::total      588
system.cpu.icache.overallMisses::total     50
system.cpu.icache.overallMissRate::total 0.078370

system.l2cache.overallHits::total          10
system.l2cache.overallMisses::total        63
system.l2cache.overallMissRate::total   0.863014
```

Interpretation: L1D/L1I show non-trivial hit rates, and the L2 miss rate looks high because the ROI is tiny (mostly **compulsory** misses at cold start). For more pronounced L2 reuse I’d warm caches, use a larger footprint, or enable a simple prefetcher—but that’s outside this minimal build demo.

**Screenshot placeholder**: _ROI stats with caches_  
`![ROI With Caches](Images/roi_l1l2.png)`

---

## 8) Troubleshooting notes I hit (and fixes)

- **“CPU has 0 interrupt controllers” fatal**: I added `system.cpu.createInterruptController()` after creating the CPU.
- **“bti unimplemented” warning**: I compiled my AArch64 binary with `-mbranch-protection=none`.
- **Failed to open binary**: I put the program next to my script and used an absolute path with:
  ```python
  here = Path(__file__).resolve().parent
  binary = str((here / "arm-hello-m5").resolve())
  ```
- **Outdir surprises**: Without `--outdir=...`, gem5 writes to `m5out/`. I passed an absolute `--outdir` to keep runs separate.

---

## 9) What I’d do next

- Swap `TimingSimpleCPU` for `O3` and compare IPC/latency sensitivity.
- Tweak cache sizes/associativity and confirm trends on a larger workload.
- Try a prefetcher: `StridePrefetcher` on L1D, `NextLinePrefetcher` on L2.

---

## 10) File layout (what this README expects)

```
configs/tutorial/part1/
├── arm_hello_m5.c
├── arm-hello-m5                # built AArch64 binary
├── caches.py                   # cache helpers
├── simple_arm.py               # no-cache config
└── two_level.py                # L1I/L1D + shared L2 config
```

---

## 11) Quick-copy commands

```bash
# Build binaries
scons -j"$(nproc)" build/ARM/gem5.opt
(cd util/m5 && scons -j"$(nproc)" build/arm64/out/m5 build/arm64/out/libm5.a)

# Build AArch64 program
aarch64-linux-gnu-g++ -O2 -mbranch-protection=none \
  -I"$PWD/include" -L"$PWD/util/m5/build/arm64/out" -lm5 \
  -o configs/tutorial/part1/arm-hello-m5 configs/tutorial/part1/arm_hello_m5.c

# Run baseline (no cache)
./build/ARM/gem5.opt configs/tutorial/part1/simple_arm.py   --outdir="$(pwd)/out/arm_nocache"

# Run with L1+L2
./build/ARM/gem5.opt configs/tutorial/part1/two_level.py    --outdir="$(pwd)/out/arm_l1l2"
```

---


# gem5 ARM SE: Minimal Build, m5-annotated Workload, and Two-Level Cache Demo

> **Goal**: I built and ran a minimal **ARM (AArch64)** system in gem5 (SE mode), then extended it to a **two-level cache** hierarchy and compared statistics. This README captures my exact steps, commands, scripts, and a few troubleshooting notes. It’s written in the first person so I can drop it straight into a repo.

---

## 0) Environment

- Distro: Ubuntu 24.04
- gem5: `v23.0.0.1` (I built from source)
- Python: 3.12 (system)
- Toolchain: `g++-aarch64-linux-gnu`
- I worked from the gem5 repo root, e.g.:

```bash
cd ~/sim/gem5-x86/gem5
```

> I used **SE (syscall emulation) mode** in this tutorial.

---

## 1) Build gem5 for ARM (SE)

I built the ARM binary (`build/ARM/gem5.opt`):

```bash
scons -j"$(nproc)" build/ARM/gem5.opt
```

If I want a multi-ISA binary instead, I can do `build/ALL/gem5.opt`, but this walkthrough uses the **ARM** build.

---

## 2) Build `m5` tool + library for ARM64

I built the `m5` helper and static library, which I link into my AArch64 test program to bracket a region-of-interest (ROI) with `m5_reset_stats`, `m5_dump_stats`, and end the sim with `m5_exit`:

```bash
cd util/m5
scons -j"$(nproc)" build/arm64/out/m5 build/arm64/out/libm5.a
cd -
```

I verified the symbols exist (optional):

```bash
nm -g util/m5/build/arm64/out/libm5.a | egrep ' m5_(reset_stats|dump_stats|exit)$'
```

---

## 3) Build a tiny **AArch64** program with `m5` ops

I created `configs/tutorial/part1/arm_hello_m5.c`:

```c
#include <stdio.h>
#include <gem5/m5ops.h>

int main() {
    m5_reset_stats(0, 0);
    printf("Hello from ARM with m5!\\n");
    m5_dump_stats(0, 0);
    m5_exit(0);
    return 0;
}
```

I compiled **for AArch64** and linked against the `m5` static library. The `-mbranch-protection=none` avoids “bti unimplemented” warnings in some gem5 builds.

> **Static** is ideal, but if my distro lacks static glibc, I can drop `-static`.

```bash
# dynamic (widely compatible):
aarch64-linux-gnu-g++ -O2 -mbranch-protection=none \
  -I"$PWD/include" \
  -L"$PWD/util/m5/build/arm64/out" -lm5 \
  -o configs/tutorial/part1/arm-hello-m5 configs/tutorial/part1/arm_hello_m5.c

# static (preferred if available):
# aarch64-linux-gnu-g++ -O2 -static -mbranch-protection=none \
#   -I"$PWD/include" \
#   -L"$PWD/util/m5/build/arm64/out" -lm5 \
#   -o configs/tutorial/part1/arm-hello-m5 configs/tutorial/part1/arm_hello_m5.c
```

Sanity check:

```bash
file configs/tutorial/part1/arm-hello-m5   # ELF 64-bit ... ARM aarch64
chmod +x configs/tutorial/part1/arm-hello-m5
```

---

## 4) **Classic** simple config (ARM, no caches)

I wrote a classic tutorial-style config `configs/tutorial/part1/simple_arm.py` (no caches: CPU → membus → DDR3).

```python
# configs/tutorial/part1/simple_arm.py
import m5
from pathlib import Path
from m5.objects import (
    System, SrcClockDomain, VoltageDomain, AddrRange,
    SystemXBar, TimingSimpleCPU,
    MemCtrl, DDR3_1600_8x8,
    Process, SEWorkload, Root
)

system = System()

system.clk_domain = SrcClockDomain()
system.clk_domain.clock = "1GHz"
system.clk_domain.voltage_domain = VoltageDomain()

system.mem_mode = "timing"
system.mem_ranges = [AddrRange("1GiB")]

system.cpu = TimingSimpleCPU()
system.cpu.createInterruptController()   # required

system.membus = SystemXBar()

# No caches: connect CPU I/D directly to membus
system.cpu.icache_port = system.membus.cpu_side_ports
system.cpu.dcache_port = system.membus.cpu_side_ports

system.system_port = system.membus.cpu_side_ports

system.mem_ctrl = MemCtrl()
system.mem_ctrl.dram = DDR3_1600_8x8()
system.mem_ctrl.dram.range = system.mem_ranges[0]
system.mem_ctrl.port = system.membus.mem_side_ports

here = Path(__file__).resolve().parent
binary = str((here / "arm-hello-m5").resolve())

system.workload = SEWorkload.init_compatible(binary)
process = Process()
process.cmd = [binary]
system.cpu.workload = process
system.cpu.createThreads()

root = Root(full_system=False, system=system)
m5.instantiate()

print("Beginning simulation!")
exit_event = m5.simulate()
print("Exiting @ tick {} because {}".format(m5.curTick(), exit_event.getCause()))
```

Run it (write to an explicit `outdir` so my artifacts don’t land in `m5out/`):

```bash
./build/ARM/gem5.opt configs/tutorial/part1/simple_arm.py --outdir="$(pwd)/out/arm_nocache"
```

**Screenshot placeholder**: _ARM simple run output_  
`![ARM Simple Run](Images/arm_simple_run.png)`

**Screenshot placeholder**: _Outdir listing (no cache)_  
`![Outdir arm_nocache](Images/arm_nocache_ls.png)`

If I forget `--outdir`, gem5 writes to `m5out/` by default.

---

## 5) Extract ROI stats (between m5_reset_stats and m5_dump_stats)

I pulled the ROI block (2nd “Begin/End Simulation Statistics”) and printed a few lines:

```bash
awk '/Begin Simulation Statistics/{b++} b==2{print} /End Simulation Statistics/{if(b==2)exit}' \
  out/arm_nocache/stats.txt > out/arm_nocache/roi_stats.txt || true
[ -s out/arm_nocache/roi_stats.txt ] || cp out/arm_nocache/stats.txt out/arm_nocache/roi_stats.txt

echo "== core metrics (no cache, ROI) =="
grep -E '^(sim_seconds|sim_ticks|sim_freq|host_inst_rate|host_op_rate)\\b' out/arm_nocache/roi_stats.txt
grep -E '^(system\\.cpu\\.)?(committedInsts|numCycles|ipc)\\b' out/arm_nocache/roi_stats.txt || true
```

**Screenshot placeholder**: _ROI stats no-cache_  
`![ROI No Cache](Images/roi_nocache.png)`

---

## 6) Add L1I/L1D + shared L2 via a small helper (`caches.py`)

I created `configs/tutorial/part1/caches.py` (parameters compatible with v23):

```python
# configs/tutorial/part1/caches.py
from m5.objects import Cache

class L1Cache(Cache):
    assoc = 2
    tag_latency = 1
    data_latency = 1
    response_latency = 1
    mshrs = 16
    tgts_per_mshr = 20

    def connectCPU(self, cpu):
        raise NotImplementedError

    def connectBus(self, bus):
        # L1 requests go "down" toward L2 via cpu_side_ports
        self.mem_side = bus.cpu_side_ports

class L1ICache(L1Cache):
    size = '16kB'
    def connectCPU(self, cpu):
        self.cpu_side = cpu.icache_port

class L1DCache(L1Cache):
    size = '64kB'
    def connectCPU(self, cpu):
        self.cpu_side = cpu.dcache_port

class L2Cache(Cache):
    size = '256kB'
    assoc = 8
    tag_latency = 10
    data_latency = 10
    response_latency = 10
    mshrs = 20
    tgts_per_mshr = 12

    def connectCPUSideBus(self, bus):
        # L2 responds "upward" to L1s via mem_side_ports
        self.cpu_side = bus.mem_side_ports

    def connectMemSideBus(self, bus):
        # L2 requests "downward" to memory via cpu_side_ports
        self.mem_side = bus.cpu_side_ports
```

Then I copied my simple config to `two_level.py` and wired the caches and an L2 crossbar:

```python
# configs/tutorial/part1/two_level.py
import m5
from pathlib import Path
from m5.objects import (
    System, SrcClockDomain, VoltageDomain, AddrRange,
    SystemXBar, L2XBar, TimingSimpleCPU,
    MemCtrl, DDR3_1600_8x8,
    Process, SEWorkload, Root
)
from caches import L1ICache, L1DCache, L2Cache

system = System()
system.clk_domain = SrcClockDomain()
system.clk_domain.clock = "1GHz"
system.clk_domain.voltage_domain = VoltageDomain()

system.mem_mode = "timing"
system.mem_ranges = [AddrRange("1GiB")]

system.cpu = TimingSimpleCPU()
system.cpu.createInterruptController()

system.membus = SystemXBar()

# L1s
system.cpu.icache = L1ICache()
system.cpu.dcache = L1DCache()
system.cpu.icache.connectCPU(system.cpu)
system.cpu.dcache.connectCPU(system.cpu)

# L2 crossbar
system.l2bus = L2XBar()
system.cpu.icache.connectBus(system.l2bus)
system.cpu.dcache.connectBus(system.l2bus)

# Shared L2
system.l2cache = L2Cache()
system.l2cache.connectCPUSideBus(system.l2bus)
system.l2cache.connectMemSideBus(system.membus)

system.system_port = system.membus.cpu_side_ports

system.mem_ctrl = MemCtrl()
system.mem_ctrl.dram = DDR3_1600_8x8()
system.mem_ctrl.dram.range = system.mem_ranges[0]
system.mem_ctrl.port = system.membus.mem_side_ports

here = Path(__file__).resolve().parent
binary = str((here / "arm-hello-m5").resolve())

system.workload = SEWorkload.init_compatible(binary)
process = Process()
process.cmd = [binary]
system.cpu.workload = process
system.cpu.createThreads()

root = Root(full_system=False, system=system)
m5.instantiate()

print("Beginning simulation (two-level caches)!")
exit_event = m5.simulate()
print("Exiting @ tick {} because {}".format(m5.curTick(), exit_event.getCause()))
```

I ran it:

```bash
./build/ARM/gem5.opt configs/tutorial/part1/two_level.py --outdir="$(pwd)/out/arm_l1l2"
```

**Screenshot placeholder**: _ARM two-level run output_  
`![ARM Two-Level Run](Images/arm_l1l2_run.png)`

---

## 7) Extract cache stats (ROI) and compare

I pulled the ROI block and grabbed key counters:

```bash
awk '/Begin Simulation Statistics/{b++} b==2{print} /End Simulation Statistics/{if(b==2)exit}' \
  out/arm_l1l2/stats.txt > out/arm_l1l2/roi_stats.txt || true
[ -s out/arm_l1l2/roi_stats.txt ] || cp out/arm_l1l2/stats.txt out/arm_l1l2/roi_stats.txt

echo "== core metrics (L1+L2, ROI) =="
grep -E '^(sim_seconds|sim_ticks|sim_freq|host_inst_rate|host_op_rate)\\b' out/arm_l1l2/roi_stats.txt
grep -E '^(system\\.cpu\\.)?(committedInsts|numCycles|ipc)\\b' out/arm_l1l2/roi_stats.txt || true

echo
echo "== caches (ROI) =="
grep -Ei '(icache|dcache|l2).*(overall_(hits|misses|miss_rate))' out/arm_l1l2/roi_stats.txt || true
grep -Ei '(icache|dcache|l2).*((overallMissRate)|(overallHits)|(overallMisses))' out/arm_l1l2/roi_stats.txt || true
```

For my “hello + m5” workload I observed (representative excerpt):

```
system.cpu.dcache.overallHits::total      211
system.cpu.dcache.overallMisses::total     23
system.cpu.dcache.overallMissRate::total 0.098291

system.cpu.icache.overallHits::total      588
system.cpu.icache.overallMisses::total     50
system.cpu.icache.overallMissRate::total 0.078370

system.l2cache.overallHits::total          10
system.l2cache.overallMisses::total        63
system.l2cache.overallMissRate::total   0.863014
```

Interpretation: L1D/L1I show non-trivial hit rates, and the L2 miss rate looks high because the ROI is extremely short (mostly **compulsory** misses).

**Screenshot placeholder**: _ROI stats with caches_  
`![ROI With Caches](Images/roi_l1l2.png)`

---

## 8) L2 size sensitivity (256 KiB → 1 MiB) and ROI stats

To check whether a larger L2 helps this tiny m5-annotated workload, I changed the L2 size in my cached config and compared ROI (`m5_reset_stats` → `m5_dump_stats`) counters.

### Change I made

In `configs/tutorial/part1/two_level.py` I set the L2 size to 1 MiB:

```python
# Shared L2 (changed from 256kB)
system.l2cache = L2Cache(size='1MB')
```

Then I re-ran to a clean outdir and extracted ROI stats:

```bash
./build/ARM/gem5.opt configs/tutorial/part1/two_level.py --outdir="$(pwd)/out/arm_l2_1mb"

awk '/Begin Simulation Statistics/{b++} b==2{print} /End Simulation Statistics/{if(b==2)exit}' \
  out/arm_l2_1mb/stats.txt > out/arm_l2_1mb/roi_stats.txt || cp out/arm_l2_1mb/stats.txt out/arm_l2_1mb/roi_stats.txt
```

### Quantitative ROI cache counters (1 MiB L2)

From `out/arm_l2_1mb/roi_stats.txt` I saw:

```
system.cpu.dcache.overallHits::total            211
system.cpu.dcache.overallMisses::total           23
system.cpu.dcache.overallMissRate::total    0.098291

system.cpu.icache.overallHits::total            588
system.cpu.icache.overallMisses::total           50
system.cpu.icache.overallMissRate::total    0.078370

system.l2cache.overallHits::total                10
system.l2cache.overallMisses::total              63
system.l2cache.overallMissRate::total       0.863014
```

### Interpretation

- L1D miss rate ≈ **9.83%** (211 hits / 23 misses) and L1I ≈ **7.84%** (588 / 50) show that the L1s are doing useful work.
- L2 miss rate ≈ **86.3%** looks high because the ROI is extremely short (“hello” + exit), so almost all L2 requests are **compulsory** (cold) misses with little chance of reuse before program termination.
- When I compared against the **256 KiB** L2 (the earlier default), the ROI cache counters and miss rates were **effectively the same** for this micro workload, confirming that increasing L2 to 1 MiB didn’t materially change behavior under a cold, short ROI.
- The meaningful win remains **no-cache → L1+L2**, which reduced ROI `sim_ticks` and produced non-trivial L1 hit rates.

---

## 9) Troubleshooting notes I hit (and fixes)

- **“CPU has 0 interrupt controllers” fatal**: I added `system.cpu.createInterruptController()` after creating the CPU.
- **“bti unimplemented” warning**: I compiled my AArch64 binary with `-mbranch-protection=none`.
- **Failed to open binary**: I put the program next to my script and used an absolute path with:
  ```python
  here = Path(__file__).resolve().parent
  binary = str((here / "arm-hello-m5").resolve())
  ```
- **Outdir surprises**: Without `--outdir=...`, gem5 writes to `m5out/`. I passed an absolute `--outdir` to keep runs separate.

---

## 10) What I’d do next

- Swap `TimingSimpleCPU` for `O3` and compare IPC/latency sensitivity.
- Tweak cache sizes/associativity and confirm trends on a larger workload.
- Try a prefetcher: `StridePrefetcher` on L1D, `NextLinePrefetcher` on L2.

---

## 11) File layout (what this README expects)

```
configs/tutorial/part1/
├── arm_hello_m5.c
├── arm-hello-m5                # built AArch64 binary
├── caches.py                   # cache helpers
├── simple_arm.py               # no-cache config
└── two_level.py                # L1I/L1D + shared L2 config
```

**Screenshot placeholders (to fill later):**
- `Images/arm_simple_run.png`
- `Images/arm_nocache_ls.png`
- `Images/roi_nocache.png`
- `Images/arm_l1l2_run.png`
- `Images/roi_l1l2.png`

---

## 12) Quick-copy commands

```bash
# Build binaries
scons -j"$(nproc)" build/ARM/gem5.opt
(cd util/m5 && scons -j"$(nproc)" build/arm64/out/m5 build/arm64/out/libm5.a)

# Build AArch64 program
aarch64-linux-gnu-g++ -O2 -mbranch-protection=none \
  -I"$PWD/include" -L"$PWD/util/m5/build/arm64/out" -lm5 \
  -o configs/tutorial/part1/arm-hello-m5 configs/tutorial/part1/arm_hello_m5.c

# Run baseline (no cache)
./build/ARM/gem5.opt configs/tutorial/part1/simple_arm.py   --outdir="$(pwd)/out/arm_nocache"

# Run with L1+L2
./build/ARM/gem5.opt configs/tutorial/part1/two_level.py    --outdir="$(pwd)/out/arm_l1l2"

# Run with L1+L2 @ 1 MiB
# (edit two_level.py to set L2Cache(size='1MB') first)
./build/ARM/gem5.opt configs/tutorial/part1/two_level.py    --outdir="$(pwd)/out/arm_l2_1mb"
```

---

## 13) License

I’m retaining the gem5 project’s original licensing for their code; my snippets herein are intended for tutorial use.



