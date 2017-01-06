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