# main함수에서 옵션 처리

mdadm

## getopt_long사용법



```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <getopt.h>

char short_options[]="-Cln";

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

