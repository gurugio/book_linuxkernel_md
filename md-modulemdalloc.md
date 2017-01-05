# md\_alloc

mdadm에서 md 장치 파일을 만들 때 open 시스템 콜을 호출하고, open 시스템 콜에서 md\_probe를 호출합니다. 그럼 md\_probe가 하는 일은 뭘까요? 바로 md\_alloc을 호출하는 일입니다.

md\_alloc은 md 장치를 만드는데 핵심적인 일을 하므로 코드를 분석해보겠습니다.

## debugging md_alloc with gdb

다음 명령으로 커널을 부팅합니다. ``-s``옵션이 있어야 gdb로 커널을 디버깅할 수 있습니다. ``-s`` 옵션은 1234번 포트를 열어서 gdb와 qemu가 연결되도록 합니다.

```
qemu-system-x86_64 -cpu host -smp 4 -kernel arch/x86/boot/bzImage \
-initrd ../qemu_initramfs/initramfs_dir/initramfs-busybox-x86.cpio.gz \
-nographic -append "console=ttyS0 earlyprintk boot_delay=5" -enable-kvm \
-drive file=./tmp/diska,if=virtio,cache=none,format=raw \
-drive file=./tmp/diskb,if=virtio,cache=none,format=raw \
-redir tcp:7777::22 -s
```

커널이 부팅되었으면 다른 터미널 창을 열어서 부팅된 커널이 있는 디렉토리에서 ``gdb vmlinux`` 명령을 실행합니다.
그리고 gdb가 실행되었으면 다음 3개의 명령을 실행합니다.
* target remote localhost:1234
* b md_alloc
* c

다음은 제 랩탑의 터미널에서 실행한 결과입니다.
```
~/work/linux-torvalds$ gdb vmlinux 
GNU gdb (Ubuntu 7.10-1ubuntu2) 7.10
Copyright (C) 2015 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from vmlinux...done.
warning: File "/home/gohkim/work/linux-torvalds/scripts/gdb/vmlinux-gdb.py" auto-loading has been declined by your `auto-load safe-path' set to "$debugdir:$datadir/auto-load".
To enable execution of this file, add
	add-auto-load-safe-path /home/gohkim/work/linux-torvalds/scripts/gdb/vmlinux-gdb.py
line to your configuration file "/home/gohkim/.gdbinit".
To completely disable this security protection, add
	set auto-load safe-path /
line to your configuration file "/home/gohkim/.gdbinit".
For more information about this security protection, see the
"Auto-loading safe path" section in the GDB manual.  E.g., run from the shell:
	info "(gdb)Auto-loading safe path"
(gdb) target remote localhost:1234
Remote debugging using localhost:1234
native_safe_halt () at ./arch/x86/include/asm/irqflags.h:50
50	}
(gdb) b md_alloc
Breakpoint 1 at 0xffffffff81614f80: file drivers/md/md.c, line 4969.
(gdb) c
Continuing.
```

그리고 커널이 부팅된 터미널에서 다음과 같이 mdadm 프로그램으로 md0 디스크를 만듭니다.
```
/ # mdadm --create /dev/md0 --level 1 --raid-devices 2 /dev/vda /dev/vdb
mdadm: /dev/vda appears to be part of a raid array:
       level=raid1 devices=2 ctime=Wed Jan  4 15:25:03 2017
mdadm: Note: this array has metadata at the start and
    may not be suitable as a boot device.  If you plan to
    store '/boot' on this device please ensure that
    your boot-loader understands md/v1.x metadata, or use
    --metadata=0.90
mdadm: /dev/vdb appears to be part of a raid array:
       level=raid1 devices=2 ctime=Wed Jan  4 15:25:03 2017
Continue creating array? y
[  295.043721] md_probe start
[  295.043955] CPU: 0 PID: 1005 Comm: mdadm Not tainted 4.4.0+ #82
[  295.044416] Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS Ubuntu-1.8.2-1ubuntu1 04/01/2014
[  295.045169]  00000000000001ff ffff88000691fb88 ffffffff8130c0df 0000000000900000
[  295.045738]  ffff88000691fba0 ffffffff816152d8 0000000000900000 ffff88000691fc00
[  295.046348]  ffffffff814e0d2e 0000000000000009 ffffffff816152c0 0000000000000000
[  295.046917] Call Trace:
[  295.047099]  [<ffffffff8130c0df>] dump_stack+0x44/0x55
[  295.047459]  [<ffffffff816152d8>] md_probe+0x18/0x40
[  295.047810]  [<ffffffff814e0d2e>] kobj_lookup+0xfe/0x160
[  295.048184]  [<ffffffff816152c0>] ? md_alloc+0x340/0x340
[  295.048558]  [<ffffffff812fd8cd>] get_gendisk+0x2d/0x100
[  295.048994]  [<ffffffff811a20d5>] blkdev_get+0x55/0x310
[  295.049357]  [<ffffffff8108d45d>] ? wake_up_bit+0x1d/0x20
[  295.049757]  [<ffffffff811a175c>] ? bdget+0x10c/0x120
[  295.050107]  [<ffffffff811a2436>] blkdev_open+0x56/0x70
[  295.050469]  [<ffffffff8116ad6a>] do_dentry_open+0x1fa/0x2f0
[  295.050904]  [<ffffffff811a23e0>] ? blkdev_get_by_dev+0x50/0x50
[  295.051329]  [<ffffffff8116c011>] vfs_open+0x51/0x60
[  295.051679]  [<ffffffff8117a34f>] path_openat+0x57f/0x1270
[  295.052061]  [<ffffffff811646bb>] ? kmem_cache_alloc+0x12b/0x130
[  295.052480]  [<ffffffff812b6875>] ? selinux_inode_alloc_security+0x35/0x90
[  295.053010]  [<ffffffff8117c029>] do_filp_open+0x79/0xd0
[  295.053404]  [<ffffffff812ae9cd>] ? security_d_instantiate+0x2d/0x50
[  295.053815]  [<ffffffff811645bf>] ? kmem_cache_alloc+0x2f/0x130
[  295.054271]  [<ffffffff8117b201>] ? getname_flags+0x51/0x1f0
[  295.054764]  [<ffffffff8118825a>] ? __alloc_fd+0x3a/0x170
[  295.055139]  [<ffffffff8116c376>] do_sys_open+0x126/0x200
[  295.055512]  [<ffffffff8116c469>] SyS_open+0x19/0x20
[  295.055884]  [<ffffffff8188f86e>] entry_SYSCALL_64_fastpath+0x12/0x71
```

그러면 커널 실행이 중단된 것을 알 수 있습니다. 그리고 gdb가 실행된 터미널에서는 다음과 같이 gdb가 다음 명령을 기다립니다.
```
(gdb) c
Continuing.

Breakpoint 1, md_alloc (dev=9437184, name=0x0 <irq_stack_union>) at drivers/md/md.c:4969
4969	{
(gdb) 
```

이제 md_alloc 함수를 gdb로 한줄씩 실행하면서 분석해볼 수 있습니다.



