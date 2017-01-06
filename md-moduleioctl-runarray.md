# ioctl-RUN_ARRAY

RUN_ARRAY는 드디어 md 디스크를 생성하는 명령입니다. do_md_run함수가 처리합니다.

```
static int md_ioctl(struct block_device *bdev, fmode_t mode,
			unsigned int cmd, unsigned long arg)
{
......
	case RUN_ARRAY:
		err = do_md_run(mddev);
		goto unlock;
```

## md_run

do_md_run에서 첫번째로 호출하는 함수는 md_run입니다. md_run은 다음과 같은 일을 합니다.
* mddev->disks 리스트에 아무런 장치가 없으면 -EINVAL 반환
* mddev->pers, mddev->sysfs_active 값이 0이 아니면 새로 생성되는 디스크가 아니므로 -EBUSY 반환
* analyze_sbs 호출: mddev에 연결된 md_rdev 디스크들의 슈퍼블럭을 읽어서 정상적인 슈퍼블럭인지 확인
* 


## bitmap_load


## md_waktup_thread


## set_capacity



## revalidate_disk


## KOBJ_CHANGE uevent