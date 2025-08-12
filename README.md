# TASK_LOWPOWER_WAIT — Linux Scheduler Extension

## Project Description

**TASK_LOWPOWER_WAIT** is a new process state in the Linux kernel designed to transition tasks into a low-power waiting mode while idle.  
Unlike standard states (`TASK_INTERRUPTIBLE`, `TASK_UNINTERRUPTIBLE`, etc.), this state integrates with power management subsystems (`cpuidle`, `CPUfreq`) and allows control through a `sysfs` interface.

The primary goal of this project is to **explore and impact key Linux kernel subsystems**, including:
- Task scheduler
- Power management (`cpuidle`, `CPUfreq` governors)
- Timer subsystems (timers, hrtimers)
- Kernel-to-userspace interaction (`sysfs`, `procfs`)
- Context switching and task migration on SMP systems

While research-focused, the project also aims to reduce power consumption in scenarios where tasks can safely enter deeper waiting states.

---

## Motivation

Modern processors—especially big.LITTLE architectures—feature multiple C-states and support dynamic frequency scaling.  
However, standard task sleep states do not explicitly signal that a task is safe to be in a deep low-power state.

**TASK_LOWPOWER_WAIT** addresses this by:
- Introducing a new flag in `task_struct` recognized by the scheduler
- Triggering the CPU to enter deeper C-states when no active tasks are running
- Allowing user and system services to manage this behavior explicitly

---

## Architecture

1. **Task Structure Changes**
   - New flag/state: `TASK_LOWPOWER_WAIT`
   - Modifications in `include/linux/sched.h` and related scheduler code

2. **Scheduler Integration**
   - Handling logic in `kernel/sched/core.c` for tasks in this state
   - Proper task migration and runqueue interaction

3. **Power Management Hooks**
   - Interaction with `cpuidle` and `CPUfreq` to select optimal C-state/P-state
   - Potential extension for big.LITTLE architectures

4. **Control Interface**
   - Sysfs node `/sys/kernel/task_lowpower` to manage task states
   - Visibility in `/proc` and via `ps` command

5. **Testing**
   - QEMU SMP environment
   - Load tests (e.g., `sysbench`, `stress-ng`)
   - Metric comparison between standard and modified scheduler

---

## Build and Testing

1. Kernel build:
   ```bash
   make defconfig
   make menuconfig  # enable CONFIG_SCHED_DEBUG, CONFIG_DEBUG_KERNEL
   make -j$(nproc)

2. Running in qemu:
```bash
qemu-system-x86_64 -kernel arch/x86/boot/bzImage \
    -append "root=/dev/sda console=ttyS0" \
    -hda rootfs.img -nographic -smp 4

```
