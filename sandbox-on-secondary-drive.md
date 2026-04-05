# Running a Sandbox Container on a Secondary Drive

How to run a distrowalled container with ALL storage (image, overlay, home) on a
dedicated path, fully isolated from the host's main podman storage.

## How It Works

The `--storage-root` flag tells distrobox-create to set up an isolated podman
storage instance at the given path. It creates a `storage.conf`, directory
structure, and automatically sets `--home`. All other distrobox commands
(enter, stop, rm, list) transparently find the container via a mapping file
at `~/.config/distrobox/storage-map`.

Everything — pulled images, writable overlay layers (installed packages, config
changes), and the home directory — lives under the storage root.

## Prerequisites

- podman (tested with 5.8.1)
- A secondary drive or separate path to store the container (btrfs recommended for snapshots)
- SELinux enabled (openSUSE Tumbleweed)

## Setup

### 1. Prepare the storage path

If using btrfs, create a subvolume (optional but recommended for snapshots):

```bash
sudo btrfs subvolume create /mnt/2-860evo-1tb/@sandbox
sudo chown $(whoami):$(whoami) /mnt/2-860evo-1tb/@sandbox
```

Any writable directory works — btrfs is not required.

### 2. Create the container

```bash
distrobox create --name sandbox --image registry.opensuse.org/opensuse/tumbleweed \
  --storage-root /mnt/2-860evo-1tb/@sandbox --hostname sandbox
```

Using `--hostname sandbox` gives the container a distinct hostname. KWin (and other
EWMH-compliant window managers) append the hostname to the titlebar of windows from
a different machine, so sandbox windows will show `<@sandbox>` in their title — a
visual indicator similar to Sandboxie's window tagging.

This automatically:
- Creates `storage/root/`, `storage/run/`, `home/` under the storage root
- Writes `storage.conf` pointing podman at those directories
- Sets `--home` to `<storage-root>/home`
- Records the mapping so enter/stop/rm/list work without extra flags

### 3. Fix SELinux labels (required after first enter)

The first `distrobox enter` pulls the image and initializes the container.
After that, the storage directory needs SELinux relabeling:

```bash
# First enter — pulls the image, may fail on SELinux
distrobox enter sandbox

# Relabel the storage for container access
sudo chcon -R -t container_file_t /mnt/2-860evo-1tb/@sandbox/storage/root/

# Now enter again — should work
distrobox enter sandbox
```

### 4. Use normally

No special flags needed for day-to-day use:

```bash
distrobox enter sandbox
distrobox stop sandbox
distrobox list          # shows containers from all storage roots
distrobox rm sandbox    # removes container, prints cleanup instructions for storage data
```

## D-Bus isolation

For init=0 containers (the default), distrowalled starts a container-local D-Bus
session bus so that D-Bus activated services work inside the container. This means:

- **Service activation works**: programs like Thunar can auto-start their D-Bus
  services (xfconfd, tumblerd, etc.) without manual workarounds.
- **File dialogs stay sandboxed**: XDG portal file pickers use the container's
  D-Bus, so they see the container's filesystem — not the host's.
- **Audio/video unaffected**: PipeWire, PulseAudio, X11, and Wayland use their
  own sockets and are not routed through D-Bus.
- **Host notifications isolated**: the container does not send desktop
  notifications to the host.

The container's D-Bus socket is at `/tmp/dbus-session-<UID>`. The profile
automatically sets `DBUS_SESSION_BUS_ADDRESS` to point to it.

`dbus-1-daemon` is automatically installed as part of the base package set
for openSUSE containers.

## Exporting applications

`distrobox-export` works in isolated containers but exports to the container's
own home directory instead of the host's, since the host filesystem is not
accessible:

```bash
distrobox-export --app thunar
# Desktop file placed in ~/exports/
# Icons copied to ~/exports/icons/
```

To add the app to your host's application menu, manually copy the desktop file
from `~/exports/` (inside the container, at `<storage-root>/home/exports/`) to
`~/.local/share/applications/` on the host.

## Identifying sandbox windows

When `--hostname sandbox` is used, KWin automatically appends `<@sandbox>` to the
titlebar of any GUI application launched from the container. This works because the
container has its own UTS namespace with a different hostname than the host, and X11
clients set `WM_CLIENT_MACHINE` to their hostname. KWin displays this suffix when it
differs from the host.

No additional scripts, color schemes, or window rules are needed.

> **Note:** KWin 6 supports per-window titlebar color schemes via `decocolor` in
> `kwinrulesrc`, but as of KWin 6.6.3 on X11 the Breeze decoration does not visually
> apply them. If this is fixed in a future release, you could add a colored titlebar
> for sandbox windows by creating a KWin window rule matching
> `clientmachine=sandbox`.

## What lives where

| What                              | Location                                          |
|-----------------------------------|---------------------------------------------------|
| Base image (read-only layers)     | `<storage-root>/storage/root/overlay/`            |
| Installed packages (overlay diff) | `<storage-root>/storage/root/overlay/`            |
| Container metadata                | `<storage-root>/storage/root/`                    |
| Home directory                    | `<storage-root>/home/`                            |
| Runtime state                     | `<storage-root>/storage/run/`                     |
| Storage config                    | `<storage-root>/storage.conf`                     |
| Container mapping                 | `~/.config/distrobox/storage-map`                 |
| Host podman storage               | Untouched                                         |

## Environment variable

You can also set the storage root via environment variable instead of the flag:

```bash
export DBX_CONTAINER_STORAGE_ROOT=/mnt/2-860evo-1tb/@sandbox
distrobox create --name sandbox --image fedora:39
```

## Important notes

- **SELinux relabeling** is needed after the first image pull. Without it, the container
  will fail with "cannot apply additional memory protection" or "Permission denied" errors
  on shared libraries.
- The storage path must NOT be on a filesystem mounted with `nosuid` or `nodev` — overlay needs these.
- **`--storage-root` is podman only.** Docker and lilipod are not supported.
- **Snapshots**: if using a btrfs subvolume, you can snapshot the entire container state:
  `sudo btrfs subvolume snapshot /mnt/2-860evo-1tb/@sandbox /mnt/2-860evo-1tb/@sandbox-snap`
- **Full cleanup**: `distrobox rm` removes the container but leaves the storage data.
  To fully remove, delete the storage root directory (or `sudo btrfs subvolume delete` if using btrfs).

## Approaches that were tested and rejected

These were tried before `--storage-root` was implemented:

| Approach | Why it didn't work |
|----------|--------------------|
| `-v /path:/:O` (overlay at root) | OCI runtime rejects overlaying on top of the container's own rootfs |
| `-v /usr:/usr:O,upperdir=...` | Source is the HOST's `/usr`, not the image's — wrong lower layer. Also kernel `overlayfs: failed to clone lowerpath` on btrfs |
| `--rootfs /path:O` with custom upperdir | `--rootfs` doesn't accept `upperdir`/`workdir` options |
| `--rootfs /path:O` | Ephemeral only — writes discarded on container removal |
| `--mount type=overlay` | podman 5.8.1 returns "invalid filesystem type" |
| Named volumes symlinked to secondary drive | Works but fragile — SELinux issues, hacky |
| Global `imagestore`/`graphroot` split | Works but affects ALL containers, not per-container |
| Manual `CONTAINERS_STORAGE_CONF` env var | Works but must be set on every command — now automated by `--storage-root` |
