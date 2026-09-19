# uwuBackGroundManager

**English** | [简体中文](./CN.md)

uwuBackGroundManager applies background behavior per app. It can freeze idle apps while preserving memory state, or improve retention for apps that must continue running.

## App modes

- **Default** leaves the app under Android's native process management.
- **Tombstone** freezes eligible processes after the app becomes idle.
- **Full** does not freeze the app. It limits OOM adjustment to the perceptible-app level and adds the app to the Device Idle allowlist.

Policies are stored per Android user and package name in `Settings.Secure`. `system_server` observes changes and applies the effective mode to processes sharing the app UID.

Tombstone tracks visible activities, foreground services, broadcasts, executing services, instrumentation, audio playback and recording, location, VPN, Binder activity, and AOSP freezer exemptions. Protected states temporarily prevent freezing. A Binder request wakes the whole UID so the app can process it; the UID can be frozen again after it becomes idle instead of following AOSP's frozen-process termination path.

Full reduces process reclamation and Doze restrictions, but it does not guarantee permanent survival. Force stop, crashes, voluntary exit, and severe memory pressure can still terminate the app.

## Freezer backend

The settings page offers Automatic, CGroup1, CGroup2, and Hybrid backends. The system detects the mounted cgroup layout and freezer support:

- Automatic selects a backend supported by the current device.
- Unsupported manual choices are disabled in the UI.
- If a stored choice becomes invalid, the framework falls back to an available backend. Tombstone freezing is skipped when no usable freezer exists.

Tombstone also requires Binder implementations for `BINDER_FREEZE`, `BINDER_GET_FROZEN_INFO`, and frozen-transaction tracking. Declaring ioctl numbers without the matching driver behavior is insufficient. Full mode does not require freezer-specific kernel interfaces.

## Recents behavior

“Ignore launcher task removal” applies only to Tombstone and Full apps. Swiping a card away removes it from Recents while allowing the task and process to remain. Force stop still terminates the app.

## Diagnostic log

The settings page can export a local diagnostic log. Entries use `[INFO]`, `[WARN]`, and `[ERROR]` labels and include build data, app policies, requested and active backends, cgroup controllers and mounts, Binder nodes, kernel freezer state, and framework events. Exporting reads local state only and does not upload the file.

## Kernel requirements

- CGroup1 requires a writable freezer controller.
- CGroup2 requires a writable `cgroup.freeze` interface.
- The Android Binder driver and userspace must use matching freeze UAPI structures and command numbers.
- Android userspace must be able to access the selected cgroup hierarchy.

This feature is inspired by [Cirno](https://github.com/Freezer-Team/Cirno.git).
