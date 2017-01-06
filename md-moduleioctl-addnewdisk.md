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




