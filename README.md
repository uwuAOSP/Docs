# uwuBackGroundManagerDocs

[English](./README.md) | [简体中文](./CN.md)

uwuBackGroundManager is a system feature that manages background policies on a per-app basis. It can freeze idle apps or improve their ability to remain alive in the background.

## Device Requirements

- The kernel supports cgroup freezer
- The Android Binder driver supports process freezing
- The Android userspace freezer is enabled and can access the freezer cgroup hierarchy

## How It Works

App modes are stored per user in `Settings.Secure`. The configuration persists after app restarts and device reboots. `system_server` listens for configuration changes and applies the selected mode to all processes under the app UID.

The controller tracks app visibility, audio playback, audio recording, location listeners, VPN connections, and Binder activity. These states temporarily prevent an app from being frozen. When the protected state ends, Tombstone mode schedules the app to be frozen again.

Tombstone mode freezes eligible processes after the app has been idle for approximately 3 seconds. After audio playback stops, it waits approximately 6 seconds. A process will not be frozen if it has a visible Activity, a foreground service, an active broadcast receiver, an executing service, Instrumentation, an explicit CPU capability, or an AOSP freezer exemption.

When a frozen Tombstone app receives a Binder request, uwuBackGroundManager temporarily unfreezes the entire app UID, avoiding AOSP's default frozen-process termination logic. After Binder has been idle for approximately 3 seconds, the UID can be frozen again.

Full mode does not use the freezer. It limits the process OOM adjustment value to the “perceptible app” level and adds the app package name and app ID to the Device Idle allowlist. This reduces process reclamation and Doze restrictions, but it does not guarantee that the app will never be terminated.

The optional “Ignore launcher task card removal” setting only applies to apps in Tombstone or Full mode. When an app is swiped away from Recents, its task card disappears from the launcher list, but the task, UI state, and processes are preserved. Force-stopping the app, app crashes, severe memory pressure, or the app exiting voluntarily will still terminate its processes.

## Required Kernel Support

### Tombstone Mode Requirements

- `CONFIG_CGROUP_FREEZER=y`  
  Provides the cgroup freezer required to pause and resume app processes. The active cgroup hierarchy must expose a writable `cgroup.freeze` interface to Android userspace.

- `CONFIG_ANDROID_BINDER_IPC=y`  
  Provides the Android Binder IPC driver used by app and system processes.

- `BINDER_FREEZE`  
  The Binder UAPI and driver must implement this ioctl. Android uses it to freeze Binder delivery to the target process, keeping the Binder state consistent with the cgroup frozen state.

- `BINDER_GET_FROZEN_INFO`  
  The Binder UAPI and driver must implement this ioctl and report synchronous and asynchronous transactions received while the process is frozen. The framework relies on this information to safely unfreeze Tombstone apps when Binder activity occurs.

- Binder frozen-transaction tracking  
  The driver must track pending synchronous transactions and asynchronous traffic while a process is frozen. Defining the ioctl numbers in the header alone is insufficient; the corresponding driver implementation is also required.

- Consistent userspace and kernel Binder interfaces  
  The ioctl structures and command numbers in the kernel must match those used by Android userspace.

Full mode does not require dedicated kernel hooks.

### Tombstone Mode

Preserves the in-memory state while preventing CPU usage when idle.

- Freezes eligible background processes after the protection delay ends
- Preserves memory and app state while the process remains alive
- Audio playback, recording, location, VPN, visibility, and Binder activity temporarily wake or protect the app
- The app may still be reclaimed by the system under high memory pressure
- Foreground processes or explicitly exempted processes may not be frozen

> Tip: After leaving a chat app, eligible processes will be frozen. When a Binder event is received, the app is temporarily unfrozen to process the event and is frozen again after becoming idle.

### Full

Suitable for apps that need to continuously perform background tasks.

- uwuBackGroundManager does not freeze the app
- The process OOM priority is raised to at least the “perceptible app” level
- The app is added to the Device Idle allowlist to reduce Doze restrictions
- The app can continue performing background tasks within the limits imposed by other Android permissions and restrictions
- The app may still exit voluntarily, crash, be force-stopped, or be terminated under severe memory pressure

> Tip: Download managers can continue performing background tasks and receive stronger process retention than in Default mode.

### Default

Default mode removes the app’s uwuBackGroundManager policy and restores Android’s native process management.

### Thanks

Many thanks to the Cirno project for inspiring this feature: https://github.com/Freezer-Team/Cirno.git
