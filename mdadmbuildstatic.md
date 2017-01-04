# mdadm 툴 빌드하기

mdadm툴은 이미 대부분의 배포판에서 지원하기때문에 일반적인 환경에서 사용하기 위해서는 굳이 빌드를 할 필요가 없습니다. 하지만 [우리가 직접 busybox를 이용해서 만든 루트파일시스템](https://github.com/gurugio/book_linuxkernel_blockdrv/blob/master/environment.md)에서 mdadm을 사용하려면 mdadm을 직접 빌드해야합니다. busybox는 mdadm 툴을 지원하지 않기 때문입니다.

## Download

mdadm의 최신 소스는 mdadm 툴의 메인테이너인 Neil Brown의 [홈페이지](http://neil.brown.name/blog/mdadm)등에서 다운받을 수 있습니다.

## build

소스를 받고 빌드하기전에 빌드 옵션을 바꿔야합니다. 우리가 busybox로 만든 루트파일시스템안에는 mdadm이 사용하는 공유 라이브러리들이 존재하지 않습니다. 따라서 mdadm이 모든 라이브러리를 정적으로 링크하도록 빌드합니다. 그러면 결국 mdadm 실행 파일만 우리가만든 루트파일시스템으로 복사하면 라이브러리에 상관없이 mdadm 프로그램을 실행할 수 있습니다.

소스에 있는 Makefile을 보면 이미 정적 링크를 위한 옵션이 들어가 있습니다. 다음과 같이 수정하고 빌드하면 mdadm 파일이 생성됩니다.
```
$ git diff
diff --git a/Makefile b/Makefile
index fd79cfb..2b9108e 100644
--- a/Makefile
+++ b/Makefile
@@ -103,7 +103,7 @@ MON_LDFLAGS += -pthread
 endif
 
 # If you want a static binary, you might uncomment these
-# LDFLAGS = -static
+LDFLAGS = -static
 # STRIP = -s
 LDLIBS=-ldl
```

우리가 만든 루트파일시스템의 sbin 디렉토리에 mdadm 파일을 복사한 후 루트파일시스템을 다시 생성합니다. 생성방법은 [다른 문서](https://github.com/gurugio/book_linuxkernel_blockdrv/blob/master/environment.md)를 참고합니다.

그리고 다음과 같이 mdadm을 실행해봐서 실행이 되는지를 확인합니다.
```
/ # ls sbin/mdadm 
sbin/mdadm
/ # mdadm
Usage: mdadm --help
  for help
/ # mdadm --help
mdadm is used for building, managing, and monitoring
Linux md devices (aka RAID arrays)
Usage: mdadm --create device options...
            Create a new array from unused devices.
       mdadm --assemble device options...
            Assemble a previously created array.
       mdadm --build device options...
            Create or assemble an array without metadata.
       mdadm --manage device options...
            make changes to an existing array.
       mdadm --misc options... devices
            report on or modify various md related devices.
       mdadm --grow options device
            resize/reshape an active array
       mdadm --incremental device
            add/remove a device to/from an array as appropriate
       mdadm --monitor options...
            Monitor one or more array for significant changes.
       mdadm device options...
            Shorthand for --manage.
Any parameter that does not start with '-' is treated as a device name
or, for --examine-bitmap, a file name.
The first such name is often the name of an md device.  Subsequent
names are often names of component devices.

 For detailed help on the above major modes use --help after the mode
 e.g.
         mdadm --assemble --help
 For general help on options use
         mdadm --help-options
/ # 
```

## qemu에 새로운 디스크 추가하기

qemu로 우리가 만든 루트파일시스템을 부팅할 때 새로운 디스크를 추가할 수 있습니다. 다음과 같이 호스트 머신에서 2개의 파일을 생성합니다. 그리고 각 파일을 qemu의 drive 옵션으로 전달하면 qemu로 부팅된 리눅스는 이 2개의 파일을 디스크로 인식합니다.

```
$ dd if=/dev/zero of=./tmp/diskb bs=1M count=100
100+0 records in
100+0 records out
104857600 bytes (105 MB) copied, 0,0543752 s, 1,9 GB/s
$ dd if=/dev/zero of=./tmp/diska bs=1M count=100
100+0 records in
100+0 records out
104857600 bytes (105 MB) copied, 0,0495879 s, 2,1 GB/s
$ cat go_tiny_qemu.sh 
qemu-system-x86_64 -cpu host -smp 4 -kernel arch/x86/boot/bzImage \
-initrd ../qemu_initramfs/initramfs_dir/initramfs-busybox-x86.cpio.gz \
-nographic -append "console=ttyS0 earlyprintk boot_delay=5" -enable-kvm \
-drive file=./tmp/diska,if=virtio,cache=none \
-drive file=./tmp/diskb,if=virtio,cache=none \
-redir tcp:7777::22 -s
```

다음과 같이 /dev 디렉토리를 보면 vda, vdb 두개의 디스크가 있는것을 확인할 수 있습니다.

```
/ # ls /dev/vd*
/dev/vda  /dev/vdb
```

다음과 같이 mdadm을 이용해서 md0 장치를 만듭니다.

```
/ # mdadm --create /dev/md0 -l 1 -n 2 /dev/vda /dev/vdb
mdadm: Note: this array has metadata at the start and
    may not be suitable as a boot device.  If you plan to
    store '/boot' on this device please ensure that
    your boot-loader understands md/v1.x metadata, or use
    --metadata=0.90
Continue creating array? y
[   49.287305] random: mdadm urandom read with 101 bits of entropy available
mdadm: Defaulting to version 1.2 metadata
[   49.306977] md: bind<vda>
[   49.308502] md: bind<vdb>
[   49.310922] md/raid1:md0: not clean -- starting background reconstruction
[   49.313083] md/raid1:md0: active with 2 out of 2 mirrors
[   49.314517] md0: detected capacity change from 0 to 104726528
mdadm: array /de[   49.316416] mdadm (1004) used greatest stack depth: 13448 bytes left
v/md0 started.
/ # [   49.318977] md: resync of RAID array md0
[   49.320068] md: minimum _guaranteed_  speed: 1000 KB/sec/disk.
[   49.321482] md: using maximum available idle IO bandwidth (but not more than 200000 KB/sec) for resync.
[   49.323462] md: using 128k window, over a total of 102272k.
[   52.499267] md: md0: resync done.
[   52.572258] md0_resync (1014) used greatest stack depth: 13280 bytes left

/ # 
/ # ls /dev/md0

/dev/md0
/ # ls /sys/block/md0
alignment_offset   ext_range          queue              slaves
bdi                holders            range              stat
capability         inflight           removable          subsystem
dev                md                 ro                 trace
discard_alignment  power              size               uevent
/ # 
```

이제 커널 소스를 분석할 준비가 끝났습니다.
