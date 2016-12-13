# md 장치 생성하는 Create 함수

main에서 프로그램 인자를 모두 읽은 후에 Create함수를 호출해서 md 장치를 생성합니다.
다음은 Create함수가 호출되는 코드입니다.
```
		rv = Create(ss, devlist->devname,
			    ident.name, ident.uuid_set ? ident.uuid : NULL,
			    devs_found-1, devlist->next,
			    &s, &c, data_offset);
```
다음은 Create 함수의 정의입니다.
```
int Create(struct supertype *st, char *mddev,
	   char *name, int *uuid,
	   int subdevs, struct mddev_dev *devlist,
	   struct shape *s,
	   struct context *c, unsigned long long data_offset)
{
```

각 인자가 어떤 값인지를 알아보기 위해 gdb로 다음과 같이 Create함수를 호출해봤습니다.

```
(gdb) b Create
Breakpoint 1 at 0x422755: file Create.c, line 69.
(gdb) r --create /dev/md0 --level 1 --raid-disks 2 /dev/loop0 /dev/loop1
Starting program: /home/gohkim/work/tools/mdadm-mainstream/mdadm --create /dev/md0 --level 1 --raid-disks 2 /dev/loop0 /dev/loop1

Breakpoint 1, Create (st=0x0, mddev=0x7fffffffe78a "/dev/md0", name=0x7fffffffe05c "", uuid=0x0, subdevs=2, 
    devlist=0x6ba040, s=0x7fffffffdf70, c=0x7fffffffdfb0, data_offset=1) at Create.c:69
69	{
(gdb) p *devlist
$1 = {devname = 0x7fffffffe7ac "/dev/loop0", disposition = 0, writemostly = 0 '\000', used = 0, data_offset = 0, 
  next = 0x6ba070}
(gdb) p *s
$2 = {raiddisks = 2, sparedisks = 0, journaldisks = 0, level = 1, layout = 65534, layout_str = 0x0, chunk = 0, 
  bitmap_chunk = 65534, bitmap_file = 0x0, assume_clean = 0, write_behind = 0, size = 0}
(gdb) p *c
$3 = {readonly = 0, runstop = 0, verbose = 0, brief = 0, force = 0, homehost = 0x7fffffffe2b0 "ws00837", 
  require_homehost = 1, prefer = 0x0, export = 0, test = 0, subarray = 0x0, update = 0x0, scan = 0, SparcAdjust = 0, 
  autof = 0, delay = 5, freeze_reshape = 0, backup_file = 0x0, invalid_backup = 0, action = 0x0, nodes = 0, 
  homecluster = 0x0}
```
결론적으로 Create함수의 인자는 다음과 같이 이해할 수 있습니다.
* st: md파일의 슈퍼블럭, 아직 생성되지 않았음
* mddev: 생성될 md 파일의 이름 /dev/md0
* name: 사용하지않음
* devlist: raid를 구성할 /dev/loop0, /dev/loop1장치들의 리스트
* s: raid 장치 구성에 필요한 옵션들
* c: 기타 옵션들



