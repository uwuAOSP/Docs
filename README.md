# Background management

Configure a background policy for each app.

## App modes

| Mode | Behavior |
| --- | --- |
| Default | Use Android's standard background management |
| Tombstone | Freeze the process when the app becomes idle, keeping its memory state |
| Full | Keep the process unfrozen, adjust its OOM priority and add it to the Device Idle allowlist |

Policies are saved per Android user and package.

## Tombstone freezing

Before freezing an idle app, the framework checks foreground activities, services, audio, location, and Binder activity. A Binder request unfreezes the app so it can handle the request; it can be frozen again when idle.

## Freezer backends

Choose **Automatic, CGroup1, CGroup2, or Hybrid**. Automatic selects an available freezer backend.

CGroup1 uses its freezer controller, while CGroup2 uses `cgroup.freeze`. Binder freezing requires kernel driver support.

## Recents

“Ignore launcher task removal” applies to Tombstone and Full. Dismissing a recent-task card can preserve the app process. Force stop still ends the app.

## Diagnostics

Export a background-management report from Settings to inspect app policies, freezer backends, cgroup and Binder state, and framework events.

Tombstone freezing is inspired by [Cirno](https://github.com/Freezer-Team/Cirno).
