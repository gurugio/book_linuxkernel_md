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


