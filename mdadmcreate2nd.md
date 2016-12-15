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

```
이런식으로 md0의 슈퍼블럭에 /dev/loop0과 /dev/loop1에 대한 정보들을 기록합니다.
그러면 나중에 md0 장치 파일에 데이터가 전달되면 커널 모듈이 /dev/loop0과 /dev/loop1 장치로 데이터를 전달해야하는데, 바로 이때 슈퍼블럭에 써진 /dev/loop0과 /dev/loop1의 장치 번호를 보고 어떤 장치로 데이터를 전달해야할지를 알수 있게됩니다.




