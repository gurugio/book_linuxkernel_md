# ioctl-RAID_VERSION

mdadm 툴은 md 디스크를 만들때 ioctl 함수를 이용해 RAID_VERSION 명령을 전달해서 md 모듈의 버전 정보를 얻습니다.

다음 코드는 mdadm 툴에서 ioctl을 호출하는 코드입니다.
```
int md_get_version(int fd)
{
	struct stat stb;
	mdu_version_t vers;

	if (fstat(fd, &stb)<0)
		return -1;
	if ((S_IFMT&stb.st_mode) != S_IFBLK)
		return -1;

	if (ioctl(fd, RAID_VERSION, &vers) == 0)
		return  (vers.major*10000) + (vers.minor*100) + vers.patchlevel;
```

ioctl에 RAID_VERSION 명령을 전달하고 커널에게 mdu_version_t 객체의 포인터를 넘겨서 커널이 버전정보를 vers 변수에 저장하도록 합니다.

다음은 md 모듈에서 ioctl 시스템콜을 처리하는 md_ioctl 함수입니다.
```
static int md_ioctl(struct block_device *bdev, fmode_t mode,
			unsigned int cmd, unsigned long arg)
{
......
	void __user *argp = (void __user *)arg;
......
	switch (cmd) {
	case RAID_VERSION:
		err = get_version(argp);
		goto out;
```

md_ioctl 함수의 cmd 인자가 바로 RAID_VERSION 값입니다. 그리고 arg 인자가 mdu_version_t 객체의 포인터입니다.

다음은 get_version 함수의 코드입니다.
```
static int get_version(void __user *arg)
{
	mdu_version_t ver;

	ver.major = MD_MAJOR_VERSION;
	ver.minor = MD_MINOR_VERSION;
	ver.patchlevel = MD_PATCHLEVEL_VERSION;

	if (copy_to_user(arg, &ver, sizeof(ver)))
		return -EFAULT;

	return 0;
}
```

MD_MAJOR_VERSION, MD_MINOR_VERSION, MD_PATCHLEVEL_VERSION 값을 mdu_version_t 객체에 저장한 후 사용자 영역의 버퍼로 복사합니다.
그러면 ioctl을 호출한 사용자는 커널이 저장한 값들을 읽을 수 있습니다.


## debug with gdb

gdb로 커널이 실행되는 것을 직접 보고 싶다면 다음처럼 gdb에서 ctrl-c를 누릅니다. 그러면 gdb의 프롬프트가 출력되는데 "b md_ioctl" 명령으로 breakpoint를 만들어줍니다.

```
(gdb) c
Continuing.
^C
Program received signal SIGINT, Interrupt.
native_safe_halt () at ./arch/x86/include/asm/irqflags.h:50
50	}
(gdb) b md_ioctl
Breakpoint 3 at 0xffffffff81619a20: file drivers/md/md.c, line 6691.
```
그리고 c 명령을 내리면 커널이 계속 동작합니다.
이때 커널이 실행되고 있는 터미널에서 mdadm 툴을 이용해서 md 디스크크를 만듭니다.
```
/ # mdadm --create /dev/md0 --level 1 --raid-devices 2 /dev/vda /dev/vdb
[66833.732136] md_probe start
[66833.732379] CPU: 0 PID: 1026 Comm: mdadm Not tainted 4.4.0+ #82
[66833.732834] Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS Ubuntu-1.8.2-1ubuntu1 04/01/2014
[66833.733558]  00000000000001ff ffff88000756fb38 ffffffff8130c0df 0000000000900000
[66833.734126]  ffff88000756fb50 ffffffff816152d8 0000000000900000 ffff88000756fbb0
[66833.734681]  ffffffff814e0d2e 0000000000000009 ffffffff816152c0 0000000000000000
[66833.735214] Call Trace:
[66833.735430]  [<ffffffff8130c0df>] dump_stack+0x44/0x55
[66833.735814]  [<ffffffff816152d8>] md_probe+0x18/0x40
[66833.736184]  [<ffffffff814e0d2e>] kobj_lookup+0xfe/0x160
[66833.736559]  [<ffffffff816152c0>] ? md_alloc+0x340/0x340
[66833.736963]  [<ffffffff812fd8cd>] get_gendisk+0x2d/0x100
[66833.737333]  [<ffffffff811a1df0>] __blkdev_get+0x120/0x3b0
[66833.737731]  [<ffffffff811a221a>] blkdev_get+0x19a/0x310
[66833.738146]  [<ffffffff812b8cb8>] ? selinux_file_open+0x98/0xd0
[66833.738556]  [<ffffffff811a2436>] blkdev_open+0x56/0x70
[66833.738945]  [<ffffffff8116ad6a>] do_dentry_open+0x1fa/0x2f0
[66833.739367]  [<ffffffff811a23e0>] ? blkdev_get_by_dev+0x50/0x50
[66833.739802]  [<ffffffff8116c011>] vfs_open+0x51/0x60
[66833.740154]  [<ffffffff8117a34f>] path_openat+0x57f/0x1270
[66833.740539]  [<ffffffff811195de>] ? unlock_page+0x5e/0x60
[66833.740942]  [<ffffffff8117c029>] do_filp_open+0x79/0xd0
[66833.741314]  [<ffffffff811645bf>] ? kmem_cache_alloc+0x2f/0x130
[66833.741751]  [<ffffffff8117b201>] ? getname_flags+0x51/0x1f0
[66833.742110]  [<ffffffff8118825a>] ? __alloc_fd+0x3a/0x170
[66833.742505]  [<ffffffff8116c376>] do_sys_open+0x126/0x200
[66833.743052]  [<ffffffff8116c469>] SyS_open+0x19/0x20
[66833.743396]  [<ffffffff8188f86e>] entry_SYSCALL_64_fastpath+0x12/0x71
[66833.744747] md_probe end

```
그러면 다음과같이 커널에서 md_ioctl가 실행될 때 gdb가 커널 실행을 멈추고 사용자 명령을 기다립니다.

```
(gdb) c
Continuing.
[Switching to Thread 4]

Breakpoint 3, md_ioctl (bdev=0xffff880006174000, mode=393247, cmd=2148272400, arg=140722013883360)
    at drivers/md/md.c:6691
6691	{
(gdb) p/x cmd
$10 = 0x800c0910
```

이 다음부터는 gdb로 사용자 어플리케이션을 디버깅하는 것과 같습니다. 커널을 한줄씩 실행할 수도 있고 원하는 변수들의 값을 확인할 수도 있습니다.
