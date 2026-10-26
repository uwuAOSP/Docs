# Background management

`uwuBackGroundManager` applies process-management policies to individual apps. It can freeze idle processes while preserving their memory, or reduce the restrictions applied to selected background apps.

## App modes

| Mode | Behavior |
| --- | --- |
| Default | Uses Android's standard background management |
| Tombstone | Freezes an eligible app after it becomes idle |
| Full | Does not freeze the app; limits OOM adjustment to the perceptible-app level and adds the app to the Device Idle allowlist |

Policies are stored per Android user and package in `Settings.Secure`. `system_server` observes policy changes and applies the configured mode to processes sharing the UID.

Full mode reduces certain process reclamation and Doze restrictions. It does not prevent force stops, crashes, voluntary exits, or termination under severe memory pressure.

## Tombstone eligibility

Before freezing an app, the framework checks visible activities, foreground services, broadcasts, executing services, instrumentation, audio playback and capture, location, VPN, Binder activity, and Android freezer exemptions.

Once the protected state ends, the app can enter the freeze queue again. If a frozen UID receives a Binder request, the framework unfreezes that UID to handle the request and can freeze it again when idle, rather than terminating the frozen process through the standard AOSP path.

## Freezer backends

The settings page offers Automatic, CGroup1, CGroup2, and Hybrid.

- Automatic detects available freezer implementations from the mounted cgroup layout.
- Unsupported manual options are disabled.
- If a selected backend is no longer available, the framework falls back to a supported backend.
- Tombstone freezing is skipped when no usable freezer backend exists.

### Kernel requirements

- CGroup1 requires a writable freezer controller.
- CGroup2 requires writable `cgroup.freeze`.
- Binder freezing requires implementations of `BINDER_FREEZE`, `BINDER_GET_FROZEN_INFO`, and frozen-transaction tracking, using UAPI definitions matching userspace.
- The Android userspace freezer must be enabled and have access to the selected hierarchy.

Declaring Binder ioctl numbers without implementing the corresponding driver behavior is insufficient. Full mode does not require these freezer-specific interfaces.

## Recents

“Ignore launcher task removal” applies only to Tombstone and Full. Dismissing a card removes it from Recents without necessarily ending its task or process. Force stop still terminates the app.

## Diagnostics

The settings page can export a local report containing build information, configured and selected backends, cgroup controllers and mounts, Binder devices, kernel freezer state, app policies, and framework events. The report uses `[INFO]`, `[WARN]`, and `[ERROR]` labels. Export does not upload the file.

This feature is inspired by [Cirno](https://github.com/Freezer-Team/Cirno).