# Distrowalled

A personal fork of [distrobox](https://github.com/89luca89/distrobox) that defaults to filesystem isolation instead of tight host integration.

> **This is an opinionated personal fork.** It has only been tested on one system (see below) and may not work on yours. Use at your own risk.

## What's different from upstream distrobox

Upstream distrobox gives containers full access to your host filesystem, home directory, and `/tmp`. Distrowalled flips those defaults:

- **Host filesystem is read-only** — `/run/host/` mounts use `ro` instead of `rw`
- **Home directory is not shared** — containers get their own home
- **`/tmp` is not shared** — each container has its own `/tmp`
- **IPC and process namespace isolation on by default** — `unshare_ipc=1`, `unshare_process=1`
- **`--privileged` replaced with `--cap-add=SYS_ADMIN`** — reduced container privileges
- **AppArmor `unconfined` removed**
- **X11 forwarding via explicit socket mount** — `/tmp/.X11-unix` is mounted read-only into the container

The container always enters its own home directory rather than trying to map the host's working directory.

## Tested on

| Component | Details |
|-----------|---------|
| **Distro** | openSUSE Tumbleweed (x86_64) |
| **Kernel** | 6.19.10-1-default |
| **CPU** | AMD Ryzen 9 5900X |
| **GPU** | NVIDIA GeForce RTX 3080 12GB (GA102) |
| **Display** | X11 |
| **Container manager** | Podman 5.8.1 (rootless) |

## Mounting host directories

Since the home directory is not shared by default, you may want to mount specific host paths into your containers.

**Mount a host directory as the container's home:**

```sh
distrobox create --name mybox --home /mnt/sandbox-home
```

This is useful if you have a dedicated partition or btrfs subvolume for a container's home directory. The path is mounted into the container and `$HOME` is set to it.

**Mount arbitrary directories:**

```sh
distrobox create --name mybox --volume /path/on/host:/path/in/container:rw
```

Multiple `--volume` flags can be passed. Use `:ro` for read-only or `:rw` for read-write. Note that apps will error (`Read-only file system`) if they try to write to `:ro` mounts.

**Overlay mounts (read host files, write to a separate layer):**

If you want the container to see host files but never modify them, you can use an overlay mount. Reads fall through to the host path, writes go to a separate upper layer:

```sh
distrobox create --name mybox \
  --additional-flags "--mount type=overlay,destination=/home/miren,lowerdir=/home/miren"
```

By default the upper (writable) layer is ephemeral and discarded when the container is removed. To persist writes to a specific location:

```sh
distrobox create --name mybox \
  --additional-flags "--mount type=overlay,destination=/home/miren,lowerdir=/home/miren,upperdir=/mnt/sandbox-upper,workdir=/mnt/sandbox-work"
```

The `upperdir` and `workdir` directories must exist before creating the container. This requires `CAP_SYS_ADMIN` inside the container, which distrowalled already grants.

**Where files live without `--home`:**

If you don't specify `--home` or mount a volume for the home directory, the container's home lives inside podman's container storage layer (typically `~/.local/share/containers/storage/overlay/`). Files there persist across stop/start but are deleted when you `distrobox rm` the container.

## Isolated storage on a secondary drive

You can put a container's entire storage — images, overlays, and home — on a separate drive or path using `--storage-root`:

```sh
distrobox create --name sandbox --image fedora:39 --storage-root /mnt/secondary/@sandbox --hostname sandbox
```

Using `--hostname sandbox` causes KWin to append `<@sandbox>` to the titlebar of GUI
apps launched from the container, making it easy to tell sandbox windows apart from
host windows.

This creates an isolated podman storage instance that doesn't touch the host's default container storage. All other commands (enter, stop, rm, list) work normally without extra flags.

See [sandbox-on-secondary-drive.md](sandbox-on-secondary-drive.md) for full setup instructions including SELinux and btrfs snapshot tips.

## Known limitations

- **Wayland is untested** and likely does not work out of the box. The X11 socket mount approach won't help on a Wayland session.
- **Only tested on the system above.** Other distros, GPUs (especially AMD/Intel), or container managers (Docker, lilipod) may have issues.
- Host tools that expect a shared home directory or writable host mounts will not work as they do in upstream distrobox.

## Upstream

All credit for distrobox goes to [89luca89](https://github.com/89luca89) and the [distrobox contributors](https://github.com/89luca89/distrobox/graphs/contributors). See the [upstream project](https://github.com/89luca89/distrobox) for documentation, support, and the original README.

## License and artwork

Same as upstream — [GPL-3.0](../COPYING.md).

Artwork and logos in this repository are from the upstream distrobox project and belong to their respective creators:

- Logo by [David Lapshin](https://github.com/daudix), previous logo by [j4ckr3d](https://github.com/j4ckr3d)
- [Cardboard Box](https://skfb.ly/6Wq6q) model by [J0Y](https://sketchfab.com/lloydrostek), licensed under [CC BY 4.0](http://creativecommons.org/licenses/by/4.0)
- [GTK Loop Animation](https://github.com/gnome-design-team/gnome-mockups/blob/master/gtk/loop6.blend) by [GNOME Project](https://www.gnome.org), licensed under [CC BY-SA 3.0](https://creativecommons.org/licenses/by-sa/3.0)
