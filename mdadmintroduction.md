mdadm 툴 분석을 시작해보겠습니다.
모든 기능을 다 알아볼 수는 없으니, mdadm 코드의 전체적인 구조와 가장 기본적인 기능인 md 장치 생성, 제거만 알아보겠습니다.

## mdadm 사용법

실제로 md장치를 생성하는 명령은 다음과 같습니다.
```
$ mdadm --create /dev/md0 --level 1 --raid-disks 2 /dev/loop0 /dev/loop1
```
각 옵션은
* create: 생성할 md장치의 이름
* level: raid level
* raid-disks: RAID를 구성할 장치들

md 장치를 제거하는 명령은 다음과 같습니다.
* stop: 제거할 장치의 이름

```
$ mdadm --stop /dev/md0
```
코드를 읽을 때 위에 설명된 옵션 중심으로 분석하겠습니다.


## main 함수

가장 먼저 main가 시작되겠지요. main에서 가장 먼저 처리하는 것은 getopt_long함수를 이용한 옵션처리입니다.

### 옵션처리

옵션 처리를 알아보기 위해 다음처럼 간단한 예제를 만들어봤습니다.

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <getopt.h>

char short_options[]="-Cl:n:";

struct option long_options[] = {
    {"create",    0, 0, 'C'},
    {"level",     1, 0, 'l'},
    {"raid-disks",1, 0, 'n'},
    {0, 0, 0, 0}
};

int main(int argc, char *argv[])
{
	int option_index;
	int opt;
	
	while ((option_index = -1) ,
	       (opt=getopt_long(argc, argv,
				short_options, long_options,
				&option_index)) != -1) {
		printf("%c %d %d %s\n", (char)opt, opt, option_index, optarg);
	}
	return 0;
}
```
이 예제는 mdadm.c 파일에 있는 main함수에서 옵션을 처리하는 부분중에 우리가 사용할 create, level, raid-disks옵션만 따로 뽑아놓은 것입니다.

create옵션은
* struct options에서 has_arg 필드가 0이므로 create옵션에 추가 파라미터가 없습니다. 즉 --create=arg 나 --create arg 같은 옵션이 없다고 설정합니다.
* 짧은 옵션은 'C'입니다. long_options에서 'C'로 지정했기 때문입니다.

level, raid-disks옵션은
* 추가 파라미터가 있습니다. struct option에서 has_arg필드가 1이므로 추가 파라미터가 있다고 설정했습니다.
* short_options에서 'l:', 'n:'처럼 옵션뒤에 ':'를 붙이는게 짧은 옵션을 지정하는 문자열에서 추가 파라미터가 있다고 설정하는 것입니다.
* 짧은 옵션은 각각 'l'과 'n'입니다.

그리고 optarg라는게 있는데, 옵션외에 문자열을 읽는 것입니다. 'l'과 'n'의 추가 옵션으로 찾아지는 "1"이나 "2"등을 가르키는 포인터입니다.

다음은 실행했을 때 동일하게 동작하는지를 본 것입니다.
```
~ $ gcc a.c
~ $ ./a.out --create /dev/md0 --level 1 --raid-disks 2 /dev/loop0 /dev/loop1
C 67 0 (null)
 1 -1 /dev/md0
l 108 1 1
n 110 2 2
 1 -1 /dev/loop0
 1 -1 /dev/loop1
~ $ ./a.out -C /dev/md0 -l 1 -n 2 /dev/loop0 /dev/loop1
C 67 -1 (null)
 1 -1 /dev/md0
l 108 -1 1
n 110 -1 2
 1 -1 /dev/loop0
 1 -1 /dev/loop1
```
짧은 옵션을 쓰면 getopt_long에서 option_index값을 반환시켜주지 못합니다. 하지만 옵션에 상관없이 opt값에는 동일한 값이 반환됩니다.

실행되는 순서대로 생각해보겠습니다.
가장 먼저 getopt_long함수는 'C'옵션을 처리한 후에 '/dev/md0'을 읽습니다. 이 문자열은 옵션에 해당하지 않으므로 getopt_long함수는 1을 반환합니다.
그리고 'l'옵션은 추가 파라미터가 있다고 했으므로, '1'문자열을 읽어서 optarg로 반환합니다.
'n'옵션도 마찬가지로 추가 파라미터가 있어서 optarg에 '2'가 반환됩니다.
'/dev/loop0'옵션과 '/dev/loop1'옵션은 옵션 리스트에 없으므로 opt갑이 1이 되고 optarg로 해당 문자열이 반환됩니다.

중요한 것은 short_options와 long_options가 동일한 옵션 처리를 하도록 만들었다는 것입니다.

그러면 이제 mdadm툴에서 어떻게 옵션을 처리하는지 보겠습니다.

다음은 'create'옵션을 처리하는 코드입니다.
```
	while ((option_index = -1) ,
	       (opt=getopt_long(argc, argv,
				shortopt, long_options,
				&option_index)) != -1) {
...
		case 'C': newmode = CREATE;
			shortopt = short_bitmap_auto_options;
			break;
...
		} else if (!mode && newmode) {
			mode = newmode;
			if (mode == MISC && devs_found) {
				pr_err("No action given for %s in --misc mode\n",
					devlist->devname);
				cont_err("Action options must come before device names\n");
				exit(2);
			}
```
'create'옵션을 만나면 mode = newmode = CREATE 값이 됩니다.

다음은 '/dev/md0' 옵션을 처리하는 코드입니다.
```
		if (opt == 1) {
			/* an undecorated option - must be a device name.
			 */

			if (devs_found > 0 && devmode == DetailPlatform) {
				pr_err("controller may only be specified once. %s ignored\n",
						optarg);
				continue;
			}

			if (devs_found > 0 && mode == MANAGE && !devmode) {
				pr_err("Must give one of -a/-r/-f for subsequent devices at %s\n", optarg);
				exit(2);
			}
			if (devs_found > 0 && mode == GROW && !devmode) {
				pr_err("Must give -a/--add for devices to add: %s\n", optarg);
				exit(2);
			}
			dv = xmalloc(sizeof(*dv));
			dv->devname = optarg;
			dv->disposition = devmode;
			dv->writemostly = writemostly;
			dv->used = 0;
			dv->next = NULL;
			*devlistend = dv;
			devlistend = &dv->next;

			devs_found++;
			continue;
		}
```
옵션 리스트에 없는 문자열이므로 opt값은 1이 됩니다. 그러면 dv객체를 만들어서 optarg 값을 읽어서 '/dev/md0'문자열을 dv객체에 저장합니다.
그리고 devlist라는 리스트에 dv를 추가합니다.

다음은 'level'옵션을 처리하는 코드입니다.
```
		case O(BUILD,'l'): /* set raid level*/
			if (s.level != UnSet) {
				pr_err("raid level may only be set once.  Second value is %s.\n", optarg);
				exit(2);
			}
			s.level = map_name(pers, optarg);
			if (s.level == UnSet) {
				pr_err("invalid raid level: %s\n",
					optarg);
				exit(2);
			}
			if (s.level != 0 && s.level != LEVEL_LINEAR && s.level != 1 &&
			    s.level != LEVEL_MULTIPATH && s.level != LEVEL_FAULTY &&
			    s.level != 10 &&
			    mode == BUILD) {
				pr_err("Raid level %s not permitted with --build.\n",
					optarg);
				exit(2);
			}
			if (s.sparedisks > 0 && s.level < 1 && s.level >= -1) {
				pr_err("raid level %s is incompatible with spare-devices setting.\n",
					optarg);
				exit(2);
			}
			ident.level = s.level;
			continue;
```
'l'옵션은 추가 파라미터가 있다고 설정했으므로 optarg값이 "1"일 것입니다. 그래서 s.level = ident.level = 1이 됩니다.


```
		case O(GROW,'n'):
		case O(CREATE,'n'):
		case O(BUILD,'n'): /* number of raid disks */
			if (s.raiddisks) {
				pr_err("raid-devices set twice: %d and %s\n",
					s.raiddisks, optarg);
				exit(2);
			}
			s.raiddisks = parse_num(optarg);
			if (s.raiddisks <= 0) {
				pr_err("invalid number of raid devices: %s\n",
					optarg);
				exit(2);
			}
			ident.raid_disks = s.raiddisks;
			continue;
```
'n'옵션도 추가 파라미터가 있으므로 optarg 포인터가 가르키는 '2'문자열을 읽어서 ident.raid_disks = s.raiddisks = 2로 저장합니다.

마지막으로 "/dev/loop0"과 "/dev/loop1" 문자열도 읽어서 dv 객체를 만들어서 devlist 리스트에 추가합니다.

이렇게 옵션에 대한 처리가 끝나면 이제 실제로 장치를 생성할 차례입니다.
다음과 같이 Create함수를 호출합니다.
```
	switch(mode) {
	case CREATE:
		if (c.delay == 0)
			c.delay = DEFAULT_BITMAP_DELAY;

		if (c.nodes) {
			if (!s.bitmap_file || strcmp(s.bitmap_file, "clustered") != 0) {
				pr_err("--nodes argument only compatible with --bitmap=clustered\n");
				rv = 1;
				break;
			}

			if (s.level != 1) {
				pr_err("--bitmap=clustered is currently supported with RAID mirror only\n");
				rv = 1;
				break;
			}
		}

		if (s.write_behind && !s.bitmap_file) {
			pr_err("write-behind mode requires a bitmap.\n");
			rv = 1;
			break;
		}
		if (s.raiddisks == 0) {
			pr_err("no raid-devices specified.\n");
			rv = 1;
			break;
		}

		rv = Create(ss, devlist->devname,
			    ident.name, ident.uuid_set ? ident.uuid : NULL,
			    devs_found-1, devlist->next,
			    &s, &c, data_offset);
		break;
```
