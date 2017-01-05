# md\_init

md\_init은 md 모듈의 시작 지점입니다. md 모듈을 커널에 정적으로 포함되도록 빌드했으므로, 커널이 부팅되면서 md\_init 함수가 호출됩니다.

md\_init은 다음 3가지를 실행합니다.

* workqueue생성
  * md, md\_misc라는 이름의 workqueue를 생성합니다.
* md\_probe 등록
  * md 장치의 주번호는 9입니다.
  * 주번호가 9인 장치파일이 생성될때마다 md\_probe함수가 호출됩니다.
* /proc/mdstat 파일 생성
  * 모든 md 디스크의 상태를 출력하는 /proc/mdstat 파일을 생성합니다.

## md workqueue

다음과 같이 "md"라는 이름의 workqueue를 생성합니다.

```
static struct workqueue_struct *md_wq;
......
    md_wq = alloc_workqueue("md", WQ_MEM_RECLAIM, 0);
    if (!md_wq)
        goto err_wq;
```

예를 들어 md\_flush\_request라는 함수는 다음과 같이 md\_wq에 mddev-&gt;flush\_work를 추가합니다.

```
void md_flush_request(struct mddev *mddev, struct bio *bio)
{
    spin_lock_irq(&mddev->lock);
    wait_event_lock_irq(mddev->sb_wait,
                !mddev->flush_bio,
                mddev->lock);
    mddev->flush_bio = bio;
    spin_unlock_irq(&mddev->lock);

    INIT_WORK(&mddev->flush_work, submit_flushes);
    queue_work(md_wq, &mddev->flush_work);
}
EXPORT_SYMBOL(md_flush_request);
```

INIT\_WORK 매크로를 통해 mddev-&gt;flush\_work에 submit_flushes 함수를 연결하고 workqueue에 등록합니다. 그러면 잠시 시간이 흐른 후에 submit\__flushes 함수가 호출됩니다.

submit\_flushes 함수의 역할은 나중에 분석하겠습니다.

## md\_probe

blk_register_region함수는 커널에 블럭 장치를 등록하는 함수입니다. 이 블럭 장치는 주번호가 9 (=MD_MAJOR)이고 부번호는 0~511 을 가질 수 있습니다. 그리고 이런 장치 번호에 해당하는 장치가 생성되면 md_probe 함수를 호출합니다.

```
	blk_register_region(MKDEV(MD_MAJOR, 0), 512, THIS_MODULE,
			    md_probe, NULL, NULL);
```

blk_register_region 함수는 kobj_map 함수를 호출해서 등록된 장치를 위한 struct probe 객체를 생성하고 커널에 등록됩니다.
다음 코드가 바로 struct probe 객체를 생성하고 커널에 등록하는 코드입니다.

```
int kobj_map(struct kobj_map *domain, dev_t dev, unsigned long range,
	     struct module *module, kobj_probe_t *probe,
	     int (*lock)(dev_t, void *), void *data)
{
	unsigned n = MAJOR(dev + range - 1) - MAJOR(dev) + 1;
	unsigned index = MAJOR(dev);
	unsigned i;
	struct probe *p;

	if (n > 255)
		n = 255;

	p = kmalloc_array(n, sizeof(struct probe), GFP_KERNEL);
	if (p == NULL)
		return -ENOMEM;

	for (i = 0; i < n; i++, p++) {
		p->owner = module;
		p->get = probe;
		p->lock = lock;
		p->dev = dev;
		p->range = range;
		p->data = data;
	}
	mutex_lock(domain->lock);
	for (i = 0, p -= n; i < n; i++, p++, index++) {
		struct probe **s = &domain->probes[index % 255];
		while (*s && (*s)->range < range)
			s = &(*s)->next;
		p->next = *s;
		*s = p;
	}
...
```

kobj_map은 등록된 장치에 필요한 갯수만큼 probe 객체를 생성하고 domain->probes 리스트에 추가합니다.

md_probe 함수가 정확히 언제 호출되는지 알아보기 위해 다음과 같이 dump_stack 함수를 md_probe 함수안에 추가하고 커널을 부팅해보겠습니다.

```
diff --git a/drivers/md/md.c b/drivers/md/md.c
index 61aacab..4ec2ace 100644
--- a/drivers/md/md.c
+++ b/drivers/md/md.c
@@ -5072,7 +5072,12 @@ static int md_alloc(dev_t dev, char *name)

 static struct kobject *md_probe(dev_t dev, int *part, void *data)
 {
+
+       pr_err("md_probe start\n");
+       dump_stack();
+       
        md_alloc(dev, NULL);
+       pr_err("md_probe end\n");
        return NULL;
 }
```

커널의 부팅 과정에서 md 장치를 만들지 않기 때문에 커널의 부팅 메세지에 md_probe가 없습니다. 그러므로 다음과 같이 md0 장치를 만들어보면 그때야 md_probe가 호출되는 것을 알 수 있습니다.

```
/ # mdadm --create /dev/md0 -l 1 -n 2 /dev/vda /dev/vdb
mdadm: /dev/vda appears to be part of a raid array:
       level=raid1 devices=2 ctime=Mon Dec 19 16:42:00 2016
mdadm: Note: this array has metadata at the start and
    may not be suitable as a boot device.  If you plan to
    store '/boot' on this device please ensure that
    your boot-loader understands md/v1.x metadata, or use
    --metadata=0.90
mdadm: /dev/vdb appears to be part of a raid array:
       level=raid1 devices=2 ctime=Mon Dec 19 16:42:00 2016
Continue creating array? y
[   48.189329] md_probe start
[   48.189550] CPU: 2 PID: 1005 Comm: mdadm Not tainted 4.4.0+ #82
[   48.190012] Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS Ubuntu-1.8.2-1ubuntu1 04/01/2014
[   48.190744]  00000000000001ff ffff880006503b88 ffffffff8130c0df 0000000000900000
[   48.191303]  ffff880006503ba0 ffffffff816152d8 0000000000900000 ffff880006503c00
[   48.191876]  ffffffff814e0d2e 0000000000000009 ffffffff816152c0 0000000000000000
[   48.192405] Call Trace:
[   48.192580]  [<ffffffff8130c0df>] dump_stack+0x44/0x55
[   48.193063]  [<ffffffff816152d8>] md_probe+0x18/0x40
[   48.193423]  [<ffffffff814e0d2e>] kobj_lookup+0xfe/0x160
[   48.193795]  [<ffffffff816152c0>] ? md_alloc+0x340/0x340
[   48.194163]  [<ffffffff812fd8cd>] get_gendisk+0x2d/0x100
[   48.194527]  [<ffffffff811a20d5>] blkdev_get+0x55/0x310
[   48.194890]  [<ffffffff8108d45d>] ? wake_up_bit+0x1d/0x20
[   48.195263]  [<ffffffff811a175c>] ? bdget+0x10c/0x120
[   48.195611]  [<ffffffff811a2436>] blkdev_open+0x56/0x70
[   48.195975]  [<ffffffff8116ad6a>] do_dentry_open+0x1fa/0x2f0
[   48.196361]  [<ffffffff811a23e0>] ? blkdev_get_by_dev+0x50/0x50
[   48.196769]  [<ffffffff8116c011>] vfs_open+0x51/0x60
[   48.197112]  [<ffffffff8117a34f>] path_openat+0x57f/0x1270
[   48.197537]  [<ffffffff811646bb>] ? kmem_cache_alloc+0x12b/0x130
[   48.197951]  [<ffffffff812b6875>] ? selinux_inode_alloc_security+0x35/0x90
[   48.198418]  [<ffffffff8117c029>] do_filp_open+0x79/0xd0
[   48.198862]  [<ffffffff812ae9cd>] ? security_d_instantiate+0x2d/0x50
[   48.199300]  [<ffffffff811645bf>] ? kmem_cache_alloc+0x2f/0x130
[   48.199657]  [<ffffffff8117b201>] ? getname_flags+0x51/0x1f0
[   48.200153]  [<ffffffff8118825a>] ? __alloc_fd+0x3a/0x170
[   48.200564]  [<ffffffff8116c376>] do_sys_open+0x126/0x200
[   48.200944]  [<ffffffff8116c469>] SyS_open+0x19/0x20
[   48.201291]  [<ffffffff8188f86e>] entry_SYSCALL_64_fastpath+0x12/0x71
[   48.202087] md_probe end
[   48.202331] random: mdadm urandom read with 73 bits of entropy available
mdadm: Defaulting to version 1.2 metadata
[   48.223550] md: bind<vda>
[   48.224128] md: bind<vdb>
[   48.224809] md/raid1:md0: not clean -- starting background reconstruction
[   48.225335] md/raid1:md0: active with 2 out of 2 mirrors
[   48.225787] md0: detected capacity change from 0 to 104726528
mdadm: array /de[   48.226312] md: resync of RAID array md0
v/md0 started.
[   48.226340] mdadm (1005) used greatest stack depth: 13408 bytes left
[   48.226938] md: minimum _guaranteed_  speed: 1000 KB/sec/disk.
[   48.227240] md: using maximum available idle IO bandwidth (but not more than 200000 KB/sec) for resync.
[   48.227734] md: using 128k window, over a total of 102272k.
/ # [   52.210333] md: md0: resync done.
```

콜스택을 보면 장치 파일을 open했을 때 kobj_lookup 함수에서 md_probe를 호출합니다.
우리가 이미 분석했듯이 mdadm 툴에서 md 장치를 만들 때 mknod 시스템콜을 써서 

```
struct kobject *kobj_lookup(struct kobj_map *domain, dev_t dev, int *index)
{
	struct kobject *kobj;
	struct probe *p;
	unsigned long best = ~0UL;

retry:
	mutex_lock(domain->lock);
	for (p = domain->probes[MAJOR(dev) % 255]; p; p = p->next) {
		struct kobject *(*probe)(dev_t, int *, void *);
		struct module *owner;
		void *data;

		if (p->dev > dev || p->dev + p->range - 1 < dev)
			continue;
		if (p->range - 1 >= best)
			break;
		if (!try_module_get(p->owner))
			continue;
		owner = p->owner;
		data = p->data;
		probe = p->get;
		best = p->range - 1;
		*index = dev - p->dev;
		if (p->lock && p->lock(dev, data) < 0) {
			module_put(owner);
			continue;
		}
		mutex_unlock(domain->lock);
		kobj = probe(dev, index, data);
```
kobj_lookup함수를 보면 커널에 등록된 장치들의 장치번호를 확인해서 현재 생성되는 장치의 장치 번호에 해당하는 probe 객체를 찾아서 p->get에 저장된 함수포인터를 호출합니다.

## /proc/mdstat

다음과 같이 proc_create 함수로 /proc/mdstat 파일을 만듭니다.

```
	proc_create("mdstat", S_IRUGO, NULL, &md_seq_fops);
```

mdstat파일을 읽을 때 호출되는 함수가 md_seq_fops에 정의되어있는데, 커널에 등록된 모든 md 장치들이 all_mddevs 리스트에 연결되어있으므로, 이 리스트에서 각 장치들을 찾아서 정보를 확인하게 됩니다.
코드만 봐도 이해할 수 있는 내용이므로 여기에서 더이상 자세히 설명하지는 않겠습니다.

