# ioctl-GET_ARRAY_INFO

다음은 md_ioctl에서 GET_ARRAY_INFO를 처리하는 코드입니다.

```
	switch (cmd) {
	case GET_ARRAY_INFO:
		if (!mddev->raid_disks && !mddev->external)
			err = -ENODEV;
		else
			err = get_array_info(mddev, argp);
		goto out;
```

mddev 객체에서 raid_disks 필드와 external 필드의 값을 확인해보겠습니다.
```
Breakpoint 3, md_ioctl (bdev=0xffff880006174000, mode=393375, cmd=2152204561, arg=140731375340208)
    at drivers/md/md.c:6691
6691	{
(gdb) p/x cmd
$19 = 0x80480911
......
(gdb) n
6697		if (!md_ioctl_valid(cmd))
(gdb) n
6732		mddev = bdev->bd_disk->private_data;
(gdb) n
(gdb) p mddev->gendisk->disk_name
$26 = "md0", '\000' <repeats 28 times>
(gdb) p mddev->raid_disks
$33 = 0
(gdb) p mddev->external
$46 = 0
```

둘다 0이므로 get_array_info 함수는 호출되지 못하고 md_ioctl은 -ENODEV 값을 반환합니다.

다음과 같이 strace를 이용해서 mdadm을 분석했을 때 GET_ARRAY_INFO가 실패했던것을 다시 확인해보세요.

```
ioctl(3, 0x80480911, 0x7ffcecff5dc0)    = -1 ENODEV (No such device)  ------------GET_ARRAY_INFO
```
