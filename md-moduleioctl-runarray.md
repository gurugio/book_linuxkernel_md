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

do_md_run calls md_run and bitmap_load and set_capacity


### md_run

do\_md\_run에서 첫번째로 호출하는 함수는 md\_run입니다. md\_run은 다음과 같은 일을 합니다.

* mddev-&gt;disks 리스트에 아무런 장치가 없으면 -EINVAL 반환
* mddev-&gt;pers, mddev-&gt;sysfs\_active 값이 0이 아니면 새로 생성되는 디스크가 아니므로 -EBUSY 반환
* analyze\_sbs 호출: mddev에 연결된 md\_rdev 디스크들의 슈퍼블럭을 읽어서 정상적인 슈퍼블럭인지 확인
* 
### bitmap\_load

### md\_waktup\_thread

### set\_capacity

### revalidate\_disk

### KOBJ\_CHANGE uevent



