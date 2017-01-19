# ioctl-RUN\_ARRAY

RUN\_ARRAY는 드디어 md 디스크를 생성하는 명령입니다. do\_md\_run함수가 처리합니다.

```
static int md_ioctl(struct block_device *bdev, fmode_t mode,
            unsigned int cmd, unsigned long arg)
{
......
    case RUN_ARRAY:
        err = do_md_run(mddev);
        goto unlock;
```

## do\_md\_run

* call md_run()
* set disk size at mddev->gendisk
* revalidate_disk
  * ``md_alloc()``: ``gendisk->fops = md_fops``
  * disk->fops->revalidate_disk = md_revalidate
  * check_disk_size_change(): print message "md100: detected capacity change from 0 to 104726528"
* throw uevent KBOJ_CHANGE for user-level event handler

### md_run

do\_md\_run에서 첫번째로 호출하는 함수는 md\_run입니다. md\_run은 다음과 같은 일을 합니다.

* mddev-&gt;disks 리스트에 아무런 장치가 없으면 -EINVAL 반환
* mddev-&gt;pers, mddev-&gt;sysfs\_active 값이 0이 아니면 새로 생성되는 디스크가 아니므로 -EBUSY 반환
* analyze\_sbs 호출: mddev에 연결된 md\_rdev 디스크들의 슈퍼블럭을 읽어서 정상적인 슈퍼블럭인지 확인
* pers = &raid1_personality
* pers->run = run() in raid1.c
  * setup_conf() creates conf object
    * conf->mirrors[0].rdev: pointer to md_rdev of /dev/vda
    * conf->mirrors[1].rdev: pointer to md_rdev of /dev/vdb
    * conf->mirrors[2,3]: for replacement, in case when vda or vdb is faulty
    * conf object keeps information of raid-disks
  * "md/raid1:md100: active with 2 out of 2 mirrors\n"
    * md100: device file name
    * 2 out of: mddev->raid_disks - mddev->degraded, if all devices are clean
    * 2 mirrors: mddev->raid_disks, the number of all raid-disks
  * mddev->thread = conf->thread: md100_raid1 thread
    * run raid1d() function
      * call md_check_recovery(): check MD_RECOVERY_NEEDED bit and do recovery or resync
    * this thread is not started yet, will start after md_run() is finished
      * see ``do_md_run(): md_wakeup_thread(mddev->thread)``
  * mddev->private = conf
* mddev->in_sync = 1
* mddev->pers = pers;
* mddev->ready = 1
* set ``MD_RECOVORY_NEEDED`` bit at mddev->recovery
  * comment in md.h: ``NEEDED:   we might need to start a resync/recover``
  * After creating md, md100_raid1 thread will start resync

#### struct md_personality

md_run:
* mddev->level = 1

```
	pers = find_pers(mddev->level, mddev->clevel);

```


raid1.c:
* raid1.ko is a module to support RAID-1
* If that module does not exist or is not loaded, find_pers would fails and md module results "md: personality for level %d is not loaded!" error.

```
static struct md_personality raid1_personality =
{
	.name		= "raid1",
	.level		= 1,
	.owner		= THIS_MODULE,
	.make_request	= make_request,
	.run		= run,
	.free		= raid1_free,
	.status		= status,
	.error_handler	= error,
	.hot_add_disk	= raid1_add_disk,
	.hot_remove_disk= raid1_remove_disk,
	.spare_active	= raid1_spare_active,
	.sync_request	= sync_request,
	.resize		= raid1_resize,
	.size		= raid1_size,
	.check_reshape	= raid1_reshape,
	.quiesce	= raid1_quiesce,
	.takeover	= raid1_takeover,
	.congested	= raid1_congested,
};

static int __init raid_init(void)
{
	return register_md_personality(&raid1_personality);
}
```

### md100_raid1 thread (raid1() in raid1.c)

IMPORTANT!! CALL generic_make_request!!!


* md_check_recover()






