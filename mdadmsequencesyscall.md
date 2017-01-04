# Sequence of System-calls of mdadm tool

## strace 툴 사용하기

mdadm 툴이 어떤 시스템 콜을 호출하는지를 파악하려면 strace 툴을 쓰면 됩니다. 다음과 같이 strace 툴을 사용할 수 있습니다.

```
root@pserver:~/test_mdadm# strace mdadm --create /dev/md0 --level 1 --raid-devices 2 /dev/loop0 /dev/loop1
execve("/sbin/mdadm", ["mdadm", "--create", "/dev/md100", "--level", "1", "--raid-devices", "2", "/dev/loop0", "/dev/loop1"], [/* 27 vars */]) = 0
brk(0)                                  = 0x1957000
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f5d932be000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
......
```

그런데 너무 많은 내용이 출력되므로 다음과 같이 간단하게 md 디스크를 만드는 mknod, open 시스템콜과 ioctl만 골라서 써놓겠습니다.
이렇게 mknod, open, ioctl 시스템콜을 따로 보는 이유는 나중에 커널 소스를 분석할 때 md 모듈에서 이런 시스템콜을 어떻게 처리하는지를 파악하기 위해서입니다.

## ioctl 명령

mdadm 소스중에 md_u.h 파일을보면 ioctl 시스템콜을 통해 md 모듈에 전달할 명령들이 정의되어 있습니다.

다음은 명령중의 일부입니다.

```
#define RAID_VERSION		_IOR (MD_MAJOR, 0x10, mdu_version_t)
#define GET_ARRAY_INFO		_IOR (MD_MAJOR, 0x11, mdu_array_info_t)
#define GET_DISK_INFO		_IOR (MD_MAJOR, 0x12, mdu_disk_info_t)
#define PRINT_RAID_DEBUG	_IO (MD_MAJOR, 0x13)
#define RAID_AUTORUN		_IO (MD_MAJOR, 0x14)
#define GET_BITMAP_FILE		_IOR (MD_MAJOR, 0x15, mdu_bitmap_file_t)

/* configuration */
#define CLEAR_ARRAY		_IO (MD_MAJOR, 0x20)
#define ADD_NEW_DISK		_IOW (MD_MAJOR, 0x21, mdu_disk_info_t)
#define HOT_REMOVE_DISK		_IO (MD_MAJOR, 0x22)
#define SET_ARRAY_INFO		_IOW (MD_MAJOR, 0x23, mdu_array_info_t)
#define SET_DISK_INFO		_IO (MD_MAJOR, 0x24)
#define WRITE_RAID_INFO		_IO (MD_MAJOR, 0x25)
#define UNPROTECT_ARRAY		_IO (MD_MAJOR, 0x26)
#define PROTECT_ARRAY		_IO (MD_MAJOR, 0x27)
#define HOT_ADD_DISK		_IO (MD_MAJOR, 0x28)
#define SET_DISK_FAULTY		_IO (MD_MAJOR, 0x29)
#define SET_BITMAP_FILE		_IOW (MD_MAJOR, 0x2b, int)

/* usage */
#define RUN_ARRAY		_IOW (MD_MAJOR, 0x30, mdu_param_t)
#define START_ARRAY		_IO (MD_MAJOR, 0x31)
#define STOP_ARRAY		_IO (MD_MAJOR, 0x32)
#define STOP_ARRAY_RO		_IO (MD_MAJOR, 0x33)
#define RESTART_ARRAY_RW	_IO (MD_MAJOR, 0x34)
```

RAID_VERSION이라는 상수는 ``_IOR(MD_MARJO, 0x10, ..)`` 이라고 정의되었습니다. 여기에서 MD_MAJOR는 0x9이고, 그 다음 값이 0x10이므로 결국 최종 값은 0x?????910이 될 것입니다. strace로 출력한 결과중에 다음과 같이 ioctl에 0x800c0910 값을 전달하는 코드가 있으므로 RAID_VERSION의 값이 0x800c0910 인 것을 알 수 있습니다.

```
ioctl(3, 0x800c0910, 0x7ffcecff5a60)    = 0
```

## md 디스크 생성

다음은 ``mdadm --create`` 명령으로 md 디스크를 생성할 때 호출되는 시스템콜입니다.

```
mknod("/dev/.tmp.md.1419:9:100", S_IFBLK|0600, makedev(9, 100)) = 0
open("/dev/.tmp.md.1419:9:100", O_RDWR|O_EXCL|O_DIRECT) = 3
unlink("/dev/.tmp.md.1419:9:100")       = 0

ioctl(3, 0x800c0910, 0x7ffcecff5a60)    = 0  -------------------------------------RAID_VERSION
ioctl(3, 0x80480911, 0x7ffcecff5dc0)    = -1 ENODEV (No such device)  ------------GET_ARRAY_INFO
ioctl(3, 0x800c0910, 0x7ffcecff5ae0)    = 0 --------------------------------------RAID_VERSION
write(2, "mdadm: Defaulting to version 1.2"..., 42mdadm: Defaulting to version 1.2 metadata
) = 42

ioctl(3, 0x800c0910, 0x7ffcecff5ae0)    = 0 --------------------------------------RAID_VERSION

ioctl(3, 0x800c0910, 0x7ffcecff59f0)    = 0 --------------------------------------RAID_VERSION
ioctl(3, 0x40480923, 0x7ffcecff5aa0)    = 0 --------------------------------------SET_ARRAY_INFO

ioctl(3, 0x40140921, 0x1957128)         = 0 --------------------------------------ADD_NEW_DISK
ioctl(3, 0x40140921, 0x19572b0)         = 0 --------------------------------------ADD_NEW_DISK
write(2, "mdadm: array /dev/md100 started."..., 33mdadm: array /dev/md100 started.
) = 33
ioctl(3, 0x400c0930, 0x7ffcecff5dc0)    = 0 --------------------------------------RUN_ARRAY
```

## md 디스크 해지

다음은 ``mdadm --stop`` 명령으로 md 디스크를 제거할 때 호출되는 시스템콜입니다.

```
open("/dev/md100", O_RDWR)              = 3
fstat(3, {st_mode=S_IFBLK|0660, st_rdev=makedev(9, 100), ...}) = 0
ioctl(3, 0x800c0910, 0x7fff2f4de220)    = 0
fstat(3, {st_mode=S_IFBLK|0660, st_rdev=makedev(9, 100), ...}) = 0
ioctl(3, 0x800c0910, 0x7fff2f4de170)    = 0
fstat(3, {st_mode=S_IFBLK|0660, st_rdev=makedev(9, 100), ...}) = 0
ioctl(3, 0x800c0910, 0x7fff2f4de170)    = 0
fstat(3, {st_mode=S_IFBLK|0660, st_rdev=makedev(9, 100), ...}) = 0
close(3)                                = 0
open("/dev/md100", O_RDONLY|O_EXCL)     = 3
fstat(3, {st_mode=S_IFBLK|0660, st_rdev=makedev(9, 100), ...}) = 0
ioctl(3, 0x800c0910, 0x7fff2f4dc180)    = 0
fstat(3, {st_mode=S_IFBLK|0660, st_rdev=makedev(9, 100), ...}) = 0
open("/sys/block/md100/md/metadata_version", O_RDONLY) = 4
read(4, "1.2\n", 1024)                  = 4
close(4)                                = 0
open("/sys/block/md100/md/level", O_RDONLY) = 4
read(4, "raid1\n", 1024)                = 6
close(4)                                = 0
ioctl(3, 0x932, 0)                      = 0
ioctl(3, BLKRRPART, 0)                  = 0
open("/sys/block/md100/uevent", O_WRONLY) = 4 ------------------------------------- send event!!!
write(4, "change", 6)                   = 6
close(4)                                = 0
stat("/dev/.udev", 0x7fff2f4de230)      = -1 ENOENT (No such file or directory)
write(2, "mdadm: stopped /dev/md100\n", 26mdadm: stopped /dev/md100
) = 26
close(3)                                = 0
```