

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

```