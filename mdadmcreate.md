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
 * raiddisks: --raid-disks 옵션에서 전달된 장치 갯수
 * level: --level 옵션에서 전달된 RAID level
* c: 기타 옵션들



```
	struct mdinfo info, *infos;
...
	memset(&info, 0, sizeof(info));
```


Create함수의 시작부분은 전달된 옵션들에 문제가 없는지를 확인합니다. 서로 충돌되는 옵션이 있으면 함수가 종료됩니다.
그리고 다음 코드에서 하위 장치을 열수있는 권한이 있는지 블럭 장치가 맞는지 등을 확인합니다.

```
	for (dv=devlist; dv && !have_container; dv=dv->next, dnum++) {
...
		dfd = open(dname, O_RDONLY);
		if (dfd < 0) {
			pr_err("cannot open %s: %s\n",
				dname, strerror(errno));
			exit(2);
		}
		if (fstat(dfd, &stb) != 0 ||
		    (stb.st_mode & S_IFMT) != S_IFBLK) {
			close(dfd);
			pr_err("%s is not a block device\n",
				dname);
			exit(2);
		}
		close(dfd);
...
```

다음은 RAID 장치의 슈퍼블럭의 포맷을 결정합니다. 어떤 포맷들이 있는지등 세부 사항은 다음 링크를 참고하세요.
* https://raid.wiki.kernel.org/index.php/RAID_superblock_formats

예를 들면 다음과 같이 mdadm 프로그램의 --metadata 옵션으로 지정할 수도 있습니다.
```
# mdadm --create /dev/md0 --metadata 1.0 <blah>
```

대표적으로 0.90과 1.0 두가지가 있는데, 우리는 위에서 따로 지정하지 않았으므로 아래의 루프를 돌면서 디폴트 포맷을 찾게됩니다.

```
		if (st == NULL) {
			/* Need to choose a default metadata, which is different
			 * depending on geometry of array.
			 */
			int i;
			char *name = "default";
			for(i=0; !st && superlist[i]; i++) {
				st = superlist[i]->match_metadata_desc(name);
				if (!st)
					continue;
				if (do_default_layout)
					s->layout = default_layout(st, s->level, c->verbose);
				switch (st->ss->validate_geometry(
						st, s->level, s->layout, s->raiddisks,
						&s->chunk, s->size*2,
						dv->data_offset, dname,
						&freesize, c->verbose > 0)) {
				case -1: /* Not valid, message printed, and not
					  * worth checking any further */
					exit(2);
					break;
				case 0: /* Geometry not valid */
					free(st);
					st = NULL;
					s->chunk = do_default_chunk ? UnSet : s->chunk;
					break;
				case 1:	/* All happy */
					break;
				}
			}
```

이 루프에서 superlist라는 배열을 검색하는데 이 배열은 util.c 파일에 아래와같이 정의되어있습니다.

```
struct superswitch *superlist[] =
{
	&super0, &super1,
	&super_ddf, &super_imsm,
	&mbr, &gpt,
	NULL };
```
이 배열은 super0, super1등의 struct superwitch 타입 객체들의 배열인데 super1은 다음과 같이 super1.c 파일에 정의되어있습니다.

```
struct superswitch super1 = {
#ifndef MDASSEMBLE
	.examine_super = examine_super1,
	.brief_examine_super = brief_examine_super1,
	.export_examine_super = export_examine_super1,
	.detail_super = detail_super1,
	.brief_detail_super = brief_detail_super1,
	.export_detail_super = export_detail_super1,
	.write_init_super = write_init_super1,
	.validate_geometry = validate_geometry1,
	.add_to_super = add_to_super1,
	.examine_badblocks = examine_badblocks_super1,
	.copy_metadata = copy_metadata1,
#endif
	.match_home = match_home1,
	.uuid_from_super = uuid_from_super1,
	.getinfo_super = getinfo_super1,
	.container_content = container_content1,
	.update_super = update_super1,
	.init_super = init_super1,
	.store_super = store_super1,
	.compare_super = compare_super1,
	.load_super = load_super1,
	.match_metadata_desc = match_metadata_desc1,
	.avail_size = avail_size1,
	.add_internal_bitmap = add_internal_bitmap1,
	.locate_bitmap = locate_bitmap1,
	.write_bitmap = write_bitmap1,
	.free_super = free_super1,
#if __BYTE_ORDER == BIG_ENDIAN
	.swapuuid = 0,
#else
	.swapuuid = 1,
#endif
	.name = "1.x",
};
```
제가 실험한 환경에서 선택된 메타데이터 포맷은 다음과 같이 1.0 버전입니다.

```
(gdb) p *st->ss
$25 = {examine_super = 0x4467eb <examine_super1>, brief_examine_super = 0x447626 <brief_examine_super1>, 
  brief_examine_subarrays = 0x0, export_examine_super = 0x447801 <export_examine_super1>, 
  examine_badblocks = 0x448336 <examine_badblocks_super1>, copy_metadata = 0x447ab8 <copy_metadata1>, 
  detail_super = 0x4480a8 <detail_super1>, brief_detail_super = 0x44820f <brief_detail_super1>, 
  export_detail_super = 0x4482b3 <export_detail_super1>, detail_platform = 0x0, export_detail_platform = 0x0, 
  uuid_from_super = 0x44863f <uuid_from_super1>, getinfo_super = 0x448693 <getinfo_super1>, getinfo_super_disks = 0x0, 
  match_home = 0x4485b9 <match_home1>, update_super = 0x448f34 <update_super1>, init_super = 0x44a060 <init_super1>, 
  add_to_super = 0x44a4bd <add_to_super1>, remove_from_super = 0x0, store_super = 0x44a716 <store_super1>, 
  write_init_super = 0x44abc6 <write_init_super1>, compare_super = 0x44b284 <compare_super1>, 
  load_super = 0x44b420 <load_super1>, load_container = 0x0, match_metadata_desc = 0x44bac8 <match_metadata_desc1>, 
  avail_size = 0x44bc49 <avail_size1>, min_acceptable_spare_size = 0x0, 
  add_internal_bitmap = 0x44bdaf <add_internal_bitmap1>, locate_bitmap = 0x44c27f <locate_bitmap1>, 
  write_bitmap = 0x44c34d <write_bitmap1>, free_super = 0x44c64f <free_super1>, 
  validate_geometry = 0x44c6d5 <validate_geometry1>, container_content = 0x448eea <container_content1>, 
  default_geometry = 0x0, kill_subarray = 0x0, update_subarray = 0x0, reshape_super = 0x0, manage_reshape = 0x0, 
  open_new = 0x0, set_array_state = 0x0, set_disk = 0x0, sync_metadata = 0x0, process_update = 0x0, prepare_update = 0x0, 
  activate_spare = 0x0, get_disk_controller_domain = 0x0, recover_backup = 0x0, validate_container = 0x0, swapuuid = 1, 
  external = 0, name = 0x48fc59 "1.x"}
```

이제 슈퍼블럭의 포맷을 결정했으니 /dev/loop0장치가 슈퍼블럭 포맷에 적합한지를 확인합니다.
```
	skip_size_check:
		if (c->runstop != 1 || c->verbose >= 0) {
			int fd = open(dname, O_RDONLY);
			if (fd <0 ) {
				pr_err("Cannot open %s: %s\n",
					dname, strerror(errno));
				fail=1;
				continue;
			}
			warn |= check_ext2(fd, dname);
			warn |= check_reiser(fd, dname);
			warn |= check_raid(fd, dname);
			if (strcmp(st->ss->name, "1.x") == 0 &&
			    st->minor_version >= 1)
				/* metadata at front */
				warn |= check_partitions(fd, dname, 0, 0);
			else if (s->level == 1 || s->level == LEVEL_CONTAINER
				    || (s->level == 0 && s->raiddisks == 1))
				/* partitions could be meaningful */
				warn |= check_partitions(fd, dname, freesize*2, s->size*2);
			else
				/* partitions cannot be meaningful */
				warn |= check_partitions(fd, dname, 0, 0);
			if (strcmp(st->ss->name, "1.x") == 0 &&
			    st->minor_version >= 1 &&
			    did_default &&
			    s->level == 1 &&
			    (warn & 1024) == 0) {
				warn |= 1024;
				pr_err("Note: this array has metadata at the start and\n"
					"    may not be suitable as a boot device.  If you plan to\n"
					"    store '/boot' on this device please ensure that\n"
					"    your boot-loader understands md/v1.x metadata, or use\n"
					"    --metadata=0.90\n");
			}
			close(fd);
```
/dev/loop0 장치를 열어서 이미 슈퍼블럭에 1.x 포맷의 메타데이터가 있는건 아닌지, 파티션이 있는 장치가 아닌지 등을 확인해서 RAID를 만들 수 있는지를 검사합니다.
이상이 없다면 /dev/loop0를 닫고, 루프를 돌아 다음 장치를 검사합니다.
다음 장치를 검사할 때는 이미 슈퍼블럭의 포맷이 결정되었으므로, 슈퍼블럭 포맷을 결정하는 코드는 건너뛰고 바로 장치를 검사합니다.



