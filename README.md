# CVE-2022-29968
Proof-of-concept exploit for CVE-2022-29968 (uninitialized memory) in the Linux Kernel, specifically the io_uring system.

The crash was found with Syzkaller. The crash was analyzed by Joseph Ravichandran and Michael Wang. The exploit was written by Joseph Ravichandran.

We found & reported this bug as part of the final project for [6.858 at MIT, Spring 2022](https://css.csail.mit.edu/6.858/2022/).

Any kernel after [3e08773c3841 ("block: switch polling to be bio based")](https://github.com/torvalds/linux/commit/3e08773c3841e9db7a520908cc2b136a77d275ff) and before [32452a3eb8b6 ("io_uring: fix uninitialized field in rw io_kiocb")](https://github.com/torvalds/linux/commit/32452a3eb8b64e01e2be717f518c0be046975b9d) should be vulnerable to this.

## Patch Commit
[Our patch commit](https://github.com/torvalds/linux/commit/32452a3eb8b64e01e2be717f518c0be046975b9d)

## Writeup
[The writeup we submitted for the 6.858 final project](https://css.csail.mit.edu/6.858/2022/projects/jravi-mi27950.pdf)

## Exploit
### Requirements
- Since we need to spray pointers to a fake bio struct, SMAP/ SMEP need to be off (or, if you have a kASLR leak, you can use that here)
- `/dev/sr0` needs to be readable by an unprivileged user

Tested in a Busybox install (without KVM) with 128M of RAM:

`$QEMU -m 128M -kernel $KERNEL -initrd $INITRD -nographic -append "console=ttyS0 nokaslr no_hash_pointers ftrace_dump_on_oops"`

initramfs `init`:

```
#!/bin/sh
/bin/busybox --install -s

# Mount required file systems (very useful if you are using ftrace/ debug features)
mount -t proc none /proc
mount -t sysfs sysfs /sys
mount -t tracefs nodev /sys/kernel/tracing
mount -t debugfs none /sys/kernel/debug
mkdir -p /tmp  && mount -t tmpfs tmpfs /tmp
mount -t devtmpfs none /dev

# Setup permissions for sr0
chmod -R 0777 /dev/sr0

# Create a temp file (used by the old userfaultfd approach)
# Not needed for the public exploit
touch /tmp/test

# Switch to non-root user
su attacker

# Run shell
exec sh

# Run shell (except ^C now works)
#exec setsid sh -c 'exec sh </dev/ttyS0 >/dev/ttyS0 2>&1'
```

Our `etc/passwd` has a non-root user (`attacker`) and a root user (`root`):

```
attacker:x:1000:1000:Linux User,,,:/home/attacker:/bin/sh
root:x:0:0:root:/tmp:/bin/sh
```

Kernel was compiled with `make defconfig` for `x86_64` with some extra tracing/ debugging features enabled.

### Running it
1. Compile with `make`
1. `./spray`
1. `./exploit`
