# QEMU User-Mode Emulation Hang During Cross-Platform Container Builds

## Problem

When building container images for foreign architectures (e.g., `linux/arm64` on an `x86_64` host), QEMU user-mode emulation processes can hang indefinitely due to an eventfd emulation bug ([QEMU #2738](https://gitlab.com/qemu-project/qemu/-/work_items/2738)). This affects any tool that uses QEMU binfmt_misc for cross-platform builds, including `podman build --platform` and direct `buildah` usage.

The hung processes enter **uninterruptible sleep** (`D` state), making them immune to `SIGTERM` and `SIGKILL`. After the user cancels the build (Ctrl+C), these zombie QEMU processes persist and consume resources indefinitely.

## QEMU Hang (repro.yml)

### Status: Could not reproduce on GitHub Actions

The QEMU eventfd race condition ([QEMU #2738](https://gitlab.com/qemu-project/qemu/-/work_items/2738)) **could not be triggered** on GitHub Actions runners. All 5 concurrent attempts across multiple runs completed successfully — the QEMU version shipped with `multiarch/qemu-user-static:latest` appears to have the fix.

The `multiarch/qemu-user-static` image only has tags up to `7.2.0`, so we cannot pin the known-affected QEMU versions (9.1.1, 9.2.0) to force the hang.

The issue is still reproducible locally on systems running affected QEMU versions (see [QEMU #2738](https://gitlab.com/qemu-project/qemu/-/work_items/2738) for affected version details).

### Root Cause

**QEMU bug**: Go 1.23 introduced changes to eventfd usage. QEMU's user-mode eventfd emulation has a race condition that causes Go compiler/linker processes to deadlock in `futex_wait_queue`. This was bisected to a specific QEMU eventfd handling path.

- **Go 1.22**: Works fine under QEMU user-mode emulation
- **Go 1.23+**: Hangs during compilation under QEMU user-mode emulation (on affected QEMU versions)
- **Affected architectures**: arm64, loongarch64, riscv64 (any foreign arch via qemu-user)

## Buildah/Podman Cleanup Gap (cleanup-gap.yml)

### Status: Confirmed — reproducible on every run

Even without the QEMU hang bug, **cancelling a cross-platform `podman build` leaves orphaned QEMU processes running**. This was confirmed on every CI run.

Filed issues:
- **Buildah**: [containers/buildah#6786](https://github.com/containers/buildah/issues/6786) — signal handler has no timeout on state polling, no cgroup kill fallback
- **Podman**: [containers/podman#28502](https://github.com/containers/podman/issues/28502) — client SIGINT doesn't cancel server-side build on podman machine

### What the cleanup-gap workflow demonstrates

1. Starts `podman build --platform linux/arm64` with a Containerfile that spawns long-running processes under QEMU
2. Sends SIGINT → SIGTERM → SIGKILL to the `podman build` process
3. Checks for surviving processes — **5 orphaned `qemu-aarch64-static` processes found every time**

The orphans are reparented to PID 1 (init) and run until manually killed:

```
PID 2670: /usr/bin/qemu-aarch64-static /bin/sh   PPid: 1  wchan: sigsuspend
PID 2683: /usr/bin/qemu-aarch64-static /bin/sleep 300  PPid: 2670  wchan: hrtimer_nanosleep
PID 2685: /usr/bin/qemu-aarch64-static /bin/sleep 300  PPid: 2670  wchan: hrtimer_nanosleep
PID 2687: /usr/bin/qemu-aarch64-static /bin/sleep 300  PPid: 2670  wchan: hrtimer_nanosleep
PID 2689: /usr/bin/qemu-aarch64-static /bin/sleep 300  PPid: 2670  wchan: hrtimer_nanosleep
```

The `sigint-podman-remote` job also demonstrates the same gap via podman's systemd socket (simulating the podman machine client/server split on macOS).

### Buildah code gap

When Ctrl+C is pressed during a build, the cleanup chain has no fallback for processes that don't exit:

#### `containers/buildah/run_common.go` lines 656-705

The signal handler receives SIGINT/SIGTERM and sends SIGKILL via OCI runtime:

```go
// line 659-664
go func() {
    for range interrupted {
        if err := kill("SIGKILL").Run(); err != nil {
            logrus.Errorf("%v sending SIGKILL", err)
        }
    }
}()
```

Then polls container state every 100ms **with no timeout**:

```go
// line 665-705
for {
    select {
    case <-time.After(100 * time.Millisecond):
        stat := exec.Command(runtime, append(options.Args, "state", containerName)...)
        // checks if StateStopped — but never times out
    }
}
```

If QEMU-user is in `D` state: SIGKILL has no effect -> runtime `state` never reports stopped -> **infinite loop**.

#### `containers/buildah/run_common.go` lines 1236-1244

Parent process forwards raw signal to child with no escalation or timeout:

```go
go func() {
    for receivedSignal := range interrupted {
        if err := cmd.Process.Signal(receivedSignal); err != nil {
            logrus.Infof("%v while attempting to forward %v to child process", err, receivedSignal)
        }
    }
}()
```

Parent blocks on `cmd.Wait()` (~line 1297) — hangs if child hangs.

#### What's Missing

| What should happen | What actually happens |
|---|---|
| Timeout on state polling loop | `time.After(100ms)` with **no deadline** |
| Detect D-state processes, force-kill cgroup | No process state check, no cgroup fallback |
| Warn user of stuck processes | Silent hang |

A fix would add a deadline to the state-polling loop and fall back to cgroup-level force kill:

```go
case <-time.After(30 * time.Second):
    logrus.Warnf("container %s did not stop after SIGKILL, force-killing cgroup", containerName)
    cgroupKill(containerName)
    return
```

## Running the repros

### QEMU hang repro (repro.yml)

```bash
# Via GitHub Actions (unlikely to trigger on current runners)
gh workflow run repro.yml

# Locally (on systems with affected QEMU versions)
podman build --platform linux/arm64 -f Containerfile.repro .

# After Ctrl+C, check for stuck processes:
podman machine ssh -- ps aux | grep qemu
podman machine ssh -- cat /proc/<PID>/wchan
# Will show: futex_wait_queue
```

### Cleanup gap repro (cleanup-gap.yml)

```bash
# Via GitHub Actions (reproduces every time)
gh workflow run cleanup-gap.yml
```

## References

- [QEMU #2738 — golang 1.23 build hangs when running under qemu-user on x86_64 host](https://gitlab.com/qemu-project/qemu/-/work_items/2738)
- [containers/buildah#6786 — Build cancellation leaves orphaned QEMU processes](https://github.com/containers/buildah/issues/6786)
- [containers/podman#28502 — SIGINT to podman build does not cancel server-side build on podman machine](https://github.com/containers/podman/issues/28502)
- [Buildah run_common.go — signal handling without timeout](https://github.com/containers/buildah/blob/main/run_common.go#L656-L705)
- [Workaround: kill-stuck-qemu shell function](https://github.com/kaovilai/dotfiles/blob/main/zsh/functions/podman-utils.zsh#L63)
