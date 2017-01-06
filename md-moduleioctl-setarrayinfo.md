# ioctl-SET_ARRAY_INFO



```
static int md_alloc(dev_t dev, char *name)
{
......
	disk->private_data = mddev;
```




```
static int md_ioctl(struct block_device *bdev, fmode_t mode,
			unsigned int cmd, unsigned long arg)
{
......
	mddev = bdev->bd_disk->private_data;

```


```
	if (cmd == SET_ARRAY_INFO) {
		mdu_array_info_t info;
		if (!arg)
			memset(&info, 0, sizeof(info));
		else if (copy_from_user(&info, argp, sizeof(info))) {
			err = -EFAULT;
			goto unlock;
		}
```


```
(gdb) p info
$36 = {major_version = 1, minor_version = 2, patch_version = 0, ctime = 0, level = 0, size = 0, 
  nr_disks = 0, raid_disks = 0, md_minor = 0, not_persistent = 0, utime = 0, state = 0, 
  active_disks = 0, working_disks = 0, failed_disks = 0, spare_disks = 0, layout = 0, 
  chunk_size = 0}
```


				
```
		if (mddev->pers) {
			err = update_array_info(mddev, &info);
			if (err) {
				printk(KERN_WARNING "md: couldn't update"
				       " array info. %d\n", err);
				goto unlock;
			}
			goto unlock;
		}
		if (!list_empty(&mddev->disks)) {
			printk(KERN_WARNING
			       "md: array %s already has disks!\n",
			       mdname(mddev));
			err = -EBUSY;
			goto unlock;
		}
		if (mddev->raid_disks) {
			printk(KERN_WARNING
			       "md: array %s already initialised!\n",
			       mdname(mddev));
			err = -EBUSY;
			goto unlock;
		}
```

아직 생성이 안된 md 디스크이기때문에 mddev 객체에는 아무 정보도 없습니다.
```
(gdb) p mddev->pers
$39 = (struct md_personality *) 0x0 <irq_stack_union>
(gdb) p mddev->raid_disks
$37 = 0
(gdb) p mddev->disks
$38 = {next = 0xffff880006832818, prev = 0xffff880006832818}
```


```
		err = set_array_info(mddev, &info);
		if (err) {
			printk(KERN_WARNING "md: couldn't set"
			       " array info. %d\n", err);
			goto unlock;
		}
		goto unlock;
	}
```

```
Breakpoint 4, md_ioctl (bdev=0xffff880006174000, mode=<optimised out>, cmd=1078462755, 
    arg=<optimised out>) at drivers/md/md.c:6828
6828			err = set_array_info(mddev, &info);
(gdb) s
set_array_info (info=<optimised out>, mddev=<optimised out>) at drivers/md/md.c:6348
6348		if (info->raid_disks == 0) {
(gdb) n
6350			if (info->major_version < 0 ||
(gdb) n
6352			    super_types[info->major_version].name == NULL) {
(gdb) n
6351			    info->major_version >= ARRAY_SIZE(super_types) ||
(gdb) n
6359			mddev->major_version = info->major_version;
(gdb) n
6360			mddev->minor_version = info->minor_version;
(gdb) n
6361			mddev->patch_version = info->patch_version;
(gdb) n
6362			mddev->persistent = !info->not_persistent;
(gdb) n
6366			mddev->ctime         = get_seconds();
(gdb) n
md_ioctl (bdev=0xffff880006174000, mode=<optimised out>, cmd=1078462755, arg=<optimised out>)
    at drivers/md/md.c:6802
6802				err = -EFAULT;
(gdb) p/x cmd
$42 = 0x40480923
(gdb) p mddev->major_version
$43 = 1
(gdb) p mddev->minor_version
$44 = 2
```

