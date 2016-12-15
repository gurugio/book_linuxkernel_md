# md 장치 생성하는 Create 함수 (continue)

계속 Create함수를 보겠습니다.

struct mdinfo 타입의 info 객체를 초기화한 후 새로 생성된 md0 장치의 슈퍼블럭 초기화에 사용합니다.
다음처럼 ss->ss->init_super 함수 포인터를 호출하는데, 이 함수는 super1.c파일에 있는 init_super1함수의 포인터입니다.
```
	struct mdinfo info, *infos;
...
	info.array.level = s->level;
	info.array.size = s->size;
	info.array.raid_disks = s->raiddisks;
	/* The kernel should *know* what md_minor we are dealing
	 * with, but it chooses to trust me instead. Sigh
	 */
	info.array.md_minor = 0;
	if (fstat(mdfd, &stb)==0)
		info.array.md_minor = minor(stb.st_rdev);
	info.array.not_persistent = 0;
...
	if (!st->ss->init_super(st, &info.array, s->size, name, c->homehost, uuid,
				data_offset))
		goto abort_locked;
```

이제 RAID장치에 슈퍼 블럭이 생성되었습니다.
그 다음은 RAID 장치에 /dev/loop0과 /dev/loop1을 연결하는 것입니다.

다음과 같이 devlist리스트에 연결할 장치들이 있습니다.
```
		for (dnum=0, raid_disk_num=0, dv = devlist ; dv ;
		     dv=(dv->next)?(dv->next):moved_disk, dnum++) {
```

다음은 /dev/loop0, /dev/loop1 파일을 열어서 major, minor정보를 읽고 md0 파일의 슈퍼 블럭에 기록하는 코드입니다.
```
						fd = open(dv->devname, O_RDWR|O_EXCL);

					if (fd < 0) {
						pr_err("failed to open %s after earlier success - aborting\n",
							dv->devname);
						goto abort_locked;
					}
					fstat(fd, &stb);
					inf->disk.major = major(stb.st_rdev);
					inf->disk.minor = minor(stb.st_rdev);
...
				if (st->ss->add_to_super(st, &inf->disk,
							 fd, dv->devname,
							 dv->data_offset)) {
					ioctl(mdfd, STOP_ARRAY, NULL);
					goto abort_locked;
				}
...
			if (st->ss->write_init_super(st)) {
				st->ss->free_super(st);
				goto abort_locked;
			}
```
이런식으로 md0의 슈퍼블럭에 /dev/loop0과 /dev/loop1에 대한 정보들을 기록합니다.
그러면 나중에 md0 장치 파일에 데이터가 전달되면 커널 모듈이 /dev/loop0과 /dev/loop1 장치로 데이터를 전달해야하는데, 바로 이때 슈퍼블럭에 써진 /dev/loop0과 /dev/loop1의 장치 번호를 보고 어떤 장치로 데이터를 전달해야할지를 알수 있게됩니다.

이렇게 슈퍼블럭에 기록을 한 후에는 add_disk함수를 호출합니다.
```
				rv = add_disk(mdfd, st, &info, inf);
```

add_disk함수는 ioctl시스템콜로 md모듈에 ADD_NEW_DISK 명령을 전달하는 함수입니다.
```
		rv = ioctl(mdfd, ADD_NEW_DISK, &info->disk);
```
이때 md 모듈은 다음과 같이 추가되는 장치들이 있음을 커널 메세지로 출력합니다.
```
$ dmesg | tail
...
[276242.320380] md: bind<loop0>
[276316.677050] md: bind<loop1>
```

마지막으로 다음과 같이 md모듈에 RUN_ARRAY 명령을 전달하면 /dev/md0 장치의 생성이 완료됩니다.
```
			if (ioctl(mdfd, RUN_ARRAY, &param)) {
				pr_err("RUN_ARRAY failed: %s\n",
				       strerror(errno));
				if (info.array.chunk_size & (info.array.chunk_size-1)) {
					cont_err("Problem may be that chunk size is not a power of 2\n");
				}
				ioctl(mdfd, STOP_ARRAY, NULL);
				goto abort;
			}
```

다음은 md 모듈에서 md0 장치의 생성을 완료하면서 출력하는 메세지입니다.
메세지의 의미에 대해서는 다음에 커널 코드를 읽어보면서 알아보겠습니다.
```
[276242.320380] md: bind<loop0>
[276316.677050] md: bind<loop1>
[276868.136164] md/raid1:md0: not clean -- starting background reconstruction
[276868.136177] md/raid1:md0: active with 2 out of 2 mirrors
[276868.136208] md0: detected capacity change from 0 to 10420224
[276868.136301] md: resync of RAID array md0
[276868.136304] md: minimum _guaranteed_  speed: 1000 KB/sec/disk.
[276868.136305] md: using maximum available idle IO bandwidth (but not more than 200000 KB/sec) for resync.
[276868.136307] md: using 128k window, over a total of 10176k.
[276868.633937] md: md0: resync done.
[276868.699410] RAID1 conf printout:
[276868.699413]  --- wd:2 rd:2
[276868.699414]  disk 0, wo:0, o:1, dev:loop0
[276868.699415]  disk 1, wo:0, o:1, dev:loop1
```
메세지만 보더라도 loop0과 loop1을 이용하여 10240224바이트 크기의 md0 장치가 생성되었다는 것을 알 수 있습니다.

마지막으로 /dev/.tmp.md.25064:9:0 파일을 닫고 Create함수를 종료합니다.
```
	close(mdfd);
```

아래는 close(mdfd)를 호출하기 전에 mdadm이 가지고있는 파일들의 리스트입니다.
```
$ sudo ls /proc/25065/fd -l
total 0
lrwx------ 1 root root 64 Dez 15 12:16 0 -> /dev/pts/2
lrwx------ 1 root root 64 Dez 15 12:16 1 -> /dev/pts/2
lrwx------ 1 root root 64 Dez 15 12:16 2 -> /dev/pts/2
lrwx------ 1 root root 64 Dez 15 12:16 4 -> /dev/.tmp.md.25065:9:0 (deleted)
```

아래는 close를 호출한 뒤의 파일 리스트입니다.
```
$ sudo ls /proc/25065/fd -l
total 0
lrwx------ 1 root root 64 Dez 15 12:16 0 -> /dev/pts/2
lrwx------ 1 root root 64 Dez 15 12:16 1 -> /dev/pts/2
lrwx------ 1 root root 64 Dez 15 12:16 2 -> /dev/pts/2
```
다음과 같이 /dev/md0을 확인해보면 분명히 파일이 생성되었다는 것을 알 수 있는데
mdadm 프로그램이 가지고있는 파일들의 리스트에 /dev/md0이 없다는 것이 이상합니다.
```
$ ls /dev/md0 -l
brw-rw---- 1 root disk 9, 0 Dez 15 15:27 /dev/md0
```

그 이유는 md 모듈에서 /dev/md0을 직접 생성하기 때문입니다.
장치 파일 커널 이벤트가 발생해야만 생성됩니다.
우리는 다음처럼 udevadm 프로그램을 통해 어떤 커널 이벤트가 발생했는지를 확인할 수 있습니다.

```
$ sudo udevadm moneitor
monitor will print the received events for:
UDEV - the event which udev sends out after rule processing
KERNEL - the kernel uevent

KERNEL[278035.153055] add      /devices/virtual/bdi/9:0 (bdi)
KERNEL[278035.153114] add      /devices/virtual/block/md0 (block)
UDEV  [278035.153615] add      /devices/virtual/bdi/9:0 (bdi)
UDEV  [278035.155075] add      /devices/virtual/block/md0 (block)
KERNEL[278729.798173] change   /devices/virtual/block/loop0 (block)
UDEV  [278729.800013] change   /devices/virtual/block/loop0 (block)
KERNEL[278729.844363] change   /devices/virtual/block/loop0 (block)
KERNEL[278729.844918] change   /devices/virtual/block/loop1 (block)
UDEV  [278729.846650] change   /devices/virtual/block/loop0 (block)
UDEV  [278729.846698] change   /devices/virtual/block/loop1 (block)
KERNEL[278729.847026] change   /devices/virtual/block/md0 (block)
UDEV  [278729.848867] change   /devices/virtual/block/md0 (block)
```
우리가 mknod를 통해 major번호가 9이고 minor번호가 0인 파일을 만들면, 커널은 md 모듈의 probe함수를 호출합니다.
md 모듈의 probe함수는 md_probe인데, 이 함수에서 struct gendisk 객체를 만들고, 객체가 생성될때 커널의 블럭 레이어에서 디스크가 생성되었다는 이벤트를 발생합니다.
그래서 udevadm 프로그램으로 커널 이벤트를 확인할 수 있었던 것입니다.
이 커널 이벤트가 발생하면 systemd나 udev데몬에서 최종적으로 장치 파일을 생성합니다.
