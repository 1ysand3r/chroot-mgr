# chroot-mgr

A minimal, safe chroot distro manager for rooted Android devices. Runs any Linux distribution inside an isolated ext4 image file without touching `/data` system-wide.

## Why image-based chroot

Most Android chroot guides remount the entire `/data` partition with `suid,dev` to make chroot work. This exposes every file under `/data` — including all app private data — to setuid privilege escalation.

`chroot-mgr` instead creates a dedicated `.img` file per distro and mounts only that with the necessary flags. The suid scope is contained to the image. `/data` is never remounted.

Each distro is a single `.img` file — easy to back up, resize, or delete cleanly.

## Requirements

- Rooted Android device (KernelSU, KernelSU-Next, or Magisk)
- Termux (from [F-Droid](https://f-droid.org/packages/com.termux/))
- `e2fsprogs` for image formatting and resize (`mke2fs`, `e2fsck`, `resize2fs`)
- A Linux rootfs tarball matching your device architecture

Check your architecture:
```sh
uname -m
# aarch64 = 64-bit ARM (most modern phones)
# armv7l  = 32-bit ARM
```

## Installation

```sh
# download
curl -o chroot-mgr https://raw.githubusercontent.com/yourname/chroot-mgr/main/chroot-mgr

# install system-wide
sudo cp chroot-mgr /data/local/bin/chroot-mgr
sudo chmod +x /data/local/bin/chroot-mgr
```

Add `/data/local/bin` to your PATH in Termux if not already:
```sh
echo 'export PATH=/data/local/bin:$PATH' >> ~/.bashrc
```

Or create a Termux wrapper to avoid typing `sudo` every time:
```sh
cat > $PREFIX/bin/chroot-mgr << 'EOF'
#!/data/data/com.termux/files/usr/bin/bash
sudo /data/local/bin/chroot-mgr "$@"
EOF
chmod +x $PREFIX/bin/chroot-mgr
```

## Getting a rootfs

**Debian** (official, from Debian Docker team):
```sh
# aarch64
wget https://github.com/debuerreotype/docker-debian-artifacts/raw/dist-arm64v8/bookworm/rootfs.tar.xz -O debian.tar.xz

# armv7
wget https://github.com/debuerreotype/docker-debian-artifacts/raw/dist-armv7/bookworm/rootfs.tar.xz -O debian.tar.xz
```

**Ubuntu Base** (official, from Canonical):
```sh
# aarch64
wget https://cdimage.ubuntu.com/ubuntu-base/releases/24.04/release/ubuntu-base-24.04.1-base-arm64.tar.gz -O ubuntu.tar.gz
```

**Alpine**:
```sh
# aarch64
wget https://dl-cdn.alpinelinux.org/alpine/v3.19/releases/aarch64/alpine-minirootfs-3.19.1-aarch64.tar.gz -O alpine.tar.gz
```

Use only official sources. Avoid third-party pre-built rootfs tarballs.

## Usage

```
chroot-mgr <command> <name> [args]

  install <name> <rootfs.tar.*> [size]   create image and extract rootfs
  login   <name> [user]                  enter chroot
  remove  <name>                         delete image and data
  resize  <name> <MB>                    grow image by N megabytes
```

### Install

```sh
# install with default 4G image
sudo chroot-mgr install alpine alpine.tar.gz

# install with custom size
sudo chroot-mgr install debian debian.tar.xz 8G
```

### Login

```sh
# login as root
sudo chroot-mgr login alpine

# login as another user
sudo chroot-mgr login debian alice
```

Multiple sessions are supported. Mounts are shared across sessions and cleaned up automatically when the last session exits.

### Remove

```sh
sudo chroot-mgr remove alpine
```

The distro must not be mounted. Exit all active sessions first.

### Resize

```sh
# add 2048MB to the image
sudo chroot-mgr resize debian 2048
```

Requires `resize2fs`. Distro must be unmounted.

## First steps after install

Fix package manager and set up a user (Debian/Ubuntu example):

```sh
sudo chroot-mgr login debian

# inside chroot
apt update && apt upgrade -y
apt install sudo vim curl
passwd root
```

## How it works

All distros live under `/data/local/chroot/`:

```
/data/local/chroot/
    alpine.img          ← ext4 image, mounted with suid,dev scoped only to it
    debian.img
    mnt/
        alpine/         ← active mount point (exists only while running)
        debian/
    .alpine.track       ← ordered list of active mounts for safe teardown
    .alpine.loop        ← loop device in use (e.g. /dev/block/loop0)
    .alpine.session     ← active session count
```

On login the image is attached to a loop device and mounted. `/sys`, `/proc`, `/dev` are bind-mounted in. `devpts` is mounted fresh (not bind-mounted) for correct pty ownership. `/dev/random` is remapped to `/dev/urandom` since Android's entropy source can block indefinitely. All mounts are recorded in a track file and unmounted in reverse order on exit.

On the last session exit everything is cleaned up automatically — mounts, loop device, and temp files.

## Security

- `/data` is never remounted. `suid,dev` applies only to the individual `.img` file.
- Each distro is fully contained in a single image file.
- Backup is `cp distro.img backup.img`. Removal is `rm distro.img`.
- Images can be integrity-checked with `e2fsck` at any time while unmounted.

## License

GPL-3.0
