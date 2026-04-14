# QEMU User-Mode Emulation Hang During Cross-Platform Container Builds

## Problem

When building container images for foreign architectures (e.g., `linux/arm64` on an `x86_64` host), QEMU user-mode emulation processes can hang indefinitely due to an eventfd emulation bug ([QEMU #2738](https://gitlab.com/qemu-project/qemu/-/work_items/2738)). This affects any tool that uses QEMU binfmt_misc for cross-platform builds, including `podman build --platform` and direct `buildah` usage.

The hung processes enter **uninterruptible sleep** (`D` state), making them immune to `SIGTERM` and `SIGKILL`. After the user cancels the build (Ctrl+C), these zombie QEMU processes persist and consume resources indefinitely.

## Root Cause

**QEMU bug**: Go 1.23 introduced changes to eventfd usage. QEMU's user-mode eventfd emulation has a race condition that causes Go compiler/linker processes to deadlock in `futex_wait_queue`. This was bisected to a specific QEMU eventfd handling path.

- **Go 1.22**: Works fine under QEMU user-mode emulation
- **Go 1.23+**: Hangs during compilation under QEMU user-mode emulation
- **Affected architectures**: arm64, loongarch64, riscv64 (any foreign arch via qemu-user)

## Buildah/Podman Cleanup Gap

When Ctrl+C is pressed during a build, the cleanup chain has no fallback for processes in uninterruptible sleep:

### `containers/buildah/run_common.go` lines 656-705

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

### `containers/buildah/run_common.go` lines 1236-1244

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

### What's Missing

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

## Reproducing

This repo includes a GitHub Actions workflow that demonstrates the hang. It builds a minimal Go 1.23 program under QEMU arm64 emulation on an x86_64 runner.

### What the repro does

1. Sets up QEMU user-mode emulation via binfmt_misc registration
2. Builds a multi-stage Containerfile that compiles Go 1.23 code for `linux/arm64` using `podman build`
3. The Go compilation triggers the eventfd deadlock in QEMU
4. A 5-minute timeout prevents the runner from hanging forever

### Running locally (macOS with podman machine)

```bash
# On macOS with podman machine (x86_64 or arm64 host building for the other arch)
podman build --platform linux/arm64 -f Containerfile.repro .

# After Ctrl+C, check for stuck processes:
podman machine ssh -- ps aux | grep qemu
podman machine ssh -- cat /proc/<PID>/wchan
# Will show: futex_wait_queue
```

### Running via GitHub Actions

```bash
# Trigger the workflow
gh workflow run repro.yml

# Watch it (will timeout after 5 minutes)
gh run list --workflow=repro.yml --limit 1
```

## References

- [QEMU #2738 — golang 1.23 build hangs when running under qemu-user on x86_64 host](https://gitlab.com/qemu-project/qemu/-/work_items/2738)
- [Buildah run_common.go — signal handling without timeout](https://github.com/containers/buildah/blob/main/run_common.go#L656-L705)
- [Workaround: kill-stuck-qemu shell function](https://github.com/kaovilai/dotfiles/blob/main/zsh/functions/podman-utils.zsh#L63)
