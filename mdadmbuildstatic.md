
```
$ git diff
diff --git a/Makefile b/Makefile
index fd79cfb..2b9108e 100644
--- a/Makefile
+++ b/Makefile
@@ -103,7 +103,7 @@ MON_LDFLAGS += -pthread
 endif
 
 # If you want a static binary, you might uncomment these
-# LDFLAGS = -static
+LDFLAGS = -static
 # STRIP = -s
 LDLIBS=-ldl
```






```
/ # ls sbin/mdadm 
sbin/mdadm
/ # mdadm
Usage: mdadm --help
  for help
/ # mdadm --help
mdadm is used for building, managing, and monitoring
Linux md devices (aka RAID arrays)
Usage: mdadm --create device options...
            Create a new array from unused devices.
       mdadm --assemble device options...
            Assemble a previously created array.
       mdadm --build device options...
            Create or assemble an array without metadata.
       mdadm --manage device options...
            make changes to an existing array.
       mdadm --misc options... devices
            report on or modify various md related devices.
       mdadm --grow options device
            resize/reshape an active array
       mdadm --incremental device
            add/remove a device to/from an array as appropriate
       mdadm --monitor options...
            Monitor one or more array for significant changes.
       mdadm device options...
            Shorthand for --manage.
Any parameter that does not start with '-' is treated as a device name
or, for --examine-bitmap, a file name.
The first such name is often the name of an md device.  Subsequent
names are often names of component devices.

 For detailed help on the above major modes use --help after the mode
 e.g.
         mdadm --assemble --help
 For general help on options use
         mdadm --help-options
/ # 
```


```
$ dd if=/dev/zero of=./tmp/diskb bs=1M count=100
100+0 records in
100+0 records out
104857600 bytes (105 MB) copied, 0,0543752 s, 1,9 GB/s
$ dd if=/dev/zero of=./tmp/diska bs=1M count=100
100+0 records in
100+0 records out
104857600 bytes (105 MB) copied, 0,0495879 s, 2,1 GB/s
$ cat go_tiny_qemu.sh 
qemu-system-x86_64 -cpu host -smp 4 -kernel arch/x86/boot/bzImage \
-initrd ../qemu_initramfs/initramfs_dir/initramfs-busybox-x86.cpio.gz \
-nographic -append "console=ttyS0 earlyprintk boot_delay=5" -enable-kvm \
-drive file=./tmp/diska,if=virtio,cache=none \
-drive file=./tmp/diskb,if=virtio,cache=none \
-redir tcp:7777::22 -s
```


```
/ # ls /dev/vd*
/dev/vda  /dev/vdb
/ # mdadm --create /dev/md0 -l 1 -n 2 /dev/vda /dev/vdb
mdadm: Note: this array has metadata at the start and
    may not be suitable as a boot device.  If you plan to
    store '/boot' on this device please ensure that
    your boot-loader understands md/v1.x metadata, or use
    --metadata=0.90
Continue creating array? y
[   49.287305] random: mdadm urandom read with 101 bits of entropy available
mdadm: Defaulting to version 1.2 metadata
[   49.306977] md: bind<vda>
[   49.308502] md: bind<vdb>
[   49.310922] md/raid1:md0: not clean -- starting background reconstruction
[   49.313083] md/raid1:md0: active with 2 out of 2 mirrors
[   49.314517] md0: detected capacity change from 0 to 104726528
mdadm: array /de[   49.316416] mdadm (1004) used greatest stack depth: 13448 bytes left
v/md0 started.
/ # [   49.318977] md: resync of RAID array md0
[   49.320068] md: minimum _guaranteed_  speed: 1000 KB/sec/disk.
[   49.321482] md: using maximum available idle IO bandwidth (but not more than 200000 KB/sec) for resync.
[   49.323462] md: using 128k window, over a total of 102272k.
[   52.499267] md: md0: resync done.
[   52.572258] md0_resync (1014) used greatest stack depth: 13280 bytes left

/ # 
/ # ls /dev/md0

/dev/md0
/ # ls /sys/block/md0
alignment_offset   ext_range          queue              slaves
bdi                holders            range              stat
capability         inflight           removable          subsystem
dev                md                 ro                 trace
discard_alignment  power              size               uevent
/ # 
```