mdadm 툴 분석을 시작해보겠습니다.
모든 기능을 다 알아볼 수는 없으니, mdadm 코드의 전체적인 구조와 가장 기본적인 기능인 md 장치 생성, 제거만 알아보겠습니다.

## mdadm 사용법

실제로 md장치를 생성하는 명령은 다음과 같습니다.
```
$ mdadm --create /dev/md0 --level 1 --raid-disks /dev/loop0 /dev/loop1
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
		printf("%c %d %d\n", (char)opt, opt, option_index);
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

결국 short_options와 long_options가 동일한 옵션 처리를 하도록 만들었습니다.

다음은 실행했을 때 동일하게 동작하는지를 본 것입니다.
```
~ $ ./a.out --create /dev/md0 --level 1 --raid-disks /dev/loop0 /dev/loop1
C 67 0
 1 -1
l 108 1
n 110 2
 1 -1
~ $ ./a.out -C /dev/md0 -l 1 -n /dev/loop0 /dev/loop1
C 67 -1
 1 -1
l 108 -1
n 110 -1
 1 -1
```
짧은 옵션을 쓰면 getopt_long에서 option_index값을 반환시켜주지 못합니다. 하지만 옵션에 상관없이 opt값에는 동일한 값이 반환됩니다.

getopt_long함수는 'C'옵션을 처리한 후에 '/dev/md0'을 읽습니다. 이 문자열은 옵션에 해당하지 않으므로 getopt_long함수는 1을 반환합니다.
그리고 'l'옵션은 추가 파라미터가 있다고 했으므로 'l'옵션 다음에는 '1'문자열을 읽지만 옵션으로 처리하지 않습니다.
그래서 'l'옵션 다음에는 1이 반환되지 않았습니다.
'n'옵션도 추가 파라미터가 있어서 '/dev/loop0'문자열을 건너뛰지만 '/dev/loop1'을 만나기때문에 1을 반환합니다.

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


