# ioctl-SET_ARRAY_INFO


mdadm 툴은 다음과 같이 SET_ARRAY_INFO 명령을 실행합니다.
```
int set_array_info(int mdfd, struct supertype *st, struct mdinfo *info)
{
......
		if ((vers % 100) >= 1) { /* can use different versions */
		mdu_array_info_t inf;
		memset(&inf, 0, sizeof(inf));
		inf.major_version = info->array.major_version;
		inf.minor_version = info->array.minor_version;
		rv = ioctl(mdfd, SET_ARRAY_INFO, &inf);
```

md 모듈에서는 다음과 같이 SET_ARRAY_INFO 명령의 처리를 시작합니다.

다음은 사용자로부터 받은 mdu_array_info_t 객체를 커널 영역의 info 변수로 복사하는 것입니다.
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

gdb를 이용해서 복사된 정보가 무엇인지 확인해보면 major_version과 minor_version이 복사된 것을 알 수 있습니다.

```
(gdb) p info
$36 = {major_version = 1, minor_version = 2, patch_version = 0, ctime = 0, level = 0, size = 0, 
  nr_disks = 0, raid_disks = 0, md_minor = 0, not_persistent = 0, utime = 0, state = 0, 
  active_disks = 0, working_disks = 0, failed_disks = 0, spare_disks = 0, layout = 0, 
  chunk_size = 0}
```

다음은 mddev가 이미 존재하던 객체인지를 확인하는 코드입니다.
				
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

우리가 만든 md0 디스크는 아직 생성이 안된 디스크이기때문에 mddev 객체에는 아무 정보도 없습니다.
다음과 같이 gdb로 확인이 가능합니다.
```
(gdb) p mddev->pers
$39 = (struct md_personality *) 0x0 <irq_stack_union>
(gdb) p mddev->raid_disks
$37 = 0
(gdb) p mddev->disks
$38 = {next = 0xffff880006832818, prev = 0xffff880006832818}
```

결국 커널에서는 다음과 같이 set_array_info 함수가 호출됩니다.
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
set_array_info함수는 다음과 같이 mddev 객체의 major_version, minor_version, patch_version 필드에 사용자로부터 받은 값을 저장하는 함수입니다.
지금은 각각 1, 2, 0이 저장됩니다.
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
```

함수가 종료된뒤 mddev객체의 major_version, minor_version의 값을 확인해봤습니다.
```
(gdb) p mddev->major_version
$43 = 1
(gdb) p mddev->minor_version
$44 = 2
```

