# ioctl-ADD_NEW_DISK


ADD_NEW_DISK 명령을 처리하는 함수는 add_new_disk입니다.
gdb를 이용해서 add_new_disk에 어떤 정보가 전달되는지 확인해보겠습니다.

```
add_new_disk (mddev=0xffff880005d00000, info=0xffff880007597de0) at drivers/md/md.c:5955
5955		dev_t dev = MKDEV(info->major,info->minor);
(gdb) p *info
$52 = {number = 0, major = 253, minor = 0, raid_disk = 0, state = 6}
```
info에는 md 디스크에 연결될 /dev/vda 디스크의 장치번호가 저장되어있습니다. ``ls -s /dev/vda`` 명령으로도 vda 디스크의 장치번호를 확인할 수 있습니다.

```
# ls -l /dev/vda
brw-rw---- 1 root disk 253, 0 Jan  4 14:17 /dev/vda
```
## md_import_device

다음은 add_new_disk 함수의 코드를 읽어보겠습니다.
```
	if (!mddev->raid_disks) {
		int err;
		/* expecting a device which has a superblock */
		rdev = md_import_device(dev, mddev->major_version, mddev->minor_version);
		if (IS_ERR(rdev)) {
			printk(KERN_WARNING
				"md: md_import_device returned %ld\n",
				PTR_ERR(rdev));
			return PTR_ERR(rdev);
		}
```

mddev->raid_disks 값이 0이면 새로 생성되는 md 디스크라는 의미입니다. 따라서 md_import_device 함수를 통해 md_rdev 객체를 만듭니다.
md_import_device의 핵심은 다음과 같이 super_types[].load_super 함수를 호출하는 것입니다.

```
static struct md_rdev *md_import_device(dev_t newdev, int super_format, int super_minor)
{
......
		err = super_types[super_format].
			load_super(rdev, NULL, super_minor);
```

md_import_device를 호출할 때 super_format 인자에는 mddev->major_version값이 전달되는데 이 값은 1입니다.
다음 테이블에서 super_types[1].load_super에 저장되는 함수는 super_1_load 인걸 알 수 있습니다. 따라서 호출되는 함수는 super_1_load입니다.

```
static struct super_type super_types[] = {
	[0] = {
		.name	= "0.90.0",
		.owner	= THIS_MODULE,
		.load_super	    = super_90_load,
		.validate_super	    = super_90_validate,
		.sync_super	    = super_90_sync,
		.rdev_size_change   = super_90_rdev_size_change,
		.allow_new_offset   = super_90_allow_new_offset,
	},
	[1] = {
		.name	= "md-1",
		.owner	= THIS_MODULE,
		.load_super	    = super_1_load,
		.validate_super	    = super_1_validate,
		.sync_super	    = super_1_sync,
		.rdev_size_change   = super_1_rdev_size_change,
		.allow_new_offset   = super_1_allow_new_offset,
	},
};
```

super_1_load 함수는 /dev/vda 디스크의 슈퍼블럭을 읽습니다. 그리고 읽어온 슈퍼블럭을 미리 할당된 rdev->sb_page 페이지에 저장합니다. 그리고 rdev 객체의 여러 필드들도 슈퍼블럭에서 읽어온 정보들로 초기화합니다. 하지만 /dev/vda 디스크는 새로 생성된 디스크로 사실상 슈퍼블럭에 쓰레기값만 들어있을 것입니다. 따라서 실제로 rdev 객체는 아직 실질적인 정보가 저장되지 못합니다.

## bind_rdev_to_array

bin_rdev_to_array함수는 다음과 같이 "md: bind<vda>" 커널 메세지를 출력하고, mddev->disks 리스트에 rdev를 추가하는 함수입니다.

```
	rdev->mddev = mddev;
	printk(KERN_INFO "md: bind<%s>\n", b);
......	
		list_add_rcu(&rdev->same_set, &mddev->disks);
......

이렇게 mddev->disks 리스트에 rdev를 추가하면 다음과 같이 find_rdev 등의 함수로 mddev에 연결된 rdev 디스크들을 찾아낼 수 있습니다.

```
static struct md_rdev *find_rdev(struct mddev *mddev, dev_t dev)
{
	struct md_rdev *rdev;

	rdev_for_each(rdev, mddev)
		if (rdev->bdev->bd_dev == dev)
			return rdev;

	return NULL;
}
```

이렇게 mddev 객체에 /dev/vda, /dev/vdb 디스크에 대한 정보를 담은 md_rdev 객체들을 추가하면 add_new_disk 함수는 완료됩니다.