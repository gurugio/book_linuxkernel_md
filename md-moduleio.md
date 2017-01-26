# io handling of md module

md_alloc() set md_make_request function as bio handler.

md_alloc():
```
	blk_queue_make_request(mddev->queue, md_make_request);
```

So bio handling of md driver is done by ``md_make_request()``.

What md_make_request does is briefly:
1. call md_make_request -> mddev->pers->make_request = raid1.c:make_request
1. update statistics information

## raid1.c:make_request

create r1bio object
```
	r1_bio = mempool_alloc(conf->r1bio_pool, GFP_NOIO);
```

The comment for r1bio says:
```
/*
 * this is our 'private' RAID1 bio.
 *
 * it contains information about what kind of IO operations were started
 * for this RAID1 operation, and about their status:
 */

struct r1bio {
```

We created RAID-1 device, so bio should be go into both of two raid-devices.
Only one bio is sent to md device.
So make_request creates two bio's and send them into each raid-devices.
That two bio's are managed in bios field of r1bio structure.

Following is the code of bios field of r1bio structure.
```
	/*
	 * if the IO is in WRITE direction, then multiple bios are used.
	 * We choose the number when they are allocated.
	 */
	struct bio		*bios[0];
	/* DO NOT PUT ANY NEW FIELDS HERE - bios array is contiguously alloced*/
};
```

Actually READ operation read only one raid-device because two raid-devices have the same data.
But WRITE operation must be done for all raid-devices.

Let's go ahead.

```
	r1_bio->master_bio = bio;
	r1_bio->sectors = bio_sectors(bio);
	r1_bio->state = 0;
	r1_bio->mddev = mddev;
	r1_bio->sector = bio->bi_iter.bi_sector;
```

r1_bio has the information about the original bio.

Next code is separated for READ and WRITE.


### READ operation


First it find a disk that READ operation will be handled

```
		rdisk = read_balance(conf, r1_bio, &max_sectors);

		if (rdisk < 0) {
			/* couldn't find anywhere to read from */
			raid_end_bio_io(r1_bio);
			return;
		}
		mirror = conf->mirrors + rdisk;
```

conf->mirrors has all raid-devices, so mirror will have pointer to rdev object.

```
		read_bio = bio_clone_mddev(bio, GFP_NOIO, mddev);
```

New bio is created from bio-pool of mddev.

```
		read_bio->bi_iter.bi_sector = r1_bio->sector +
			mirror->rdev->data_offset;
		read_bio->bi_bdev = mirror->rdev->bdev;
		read_bio->bi_end_io = raid1_end_read_request;
		read_bio->bi_rw = READ | do_sync;
		read_bio->bi_private = r1_bio;
```
After bio is completed, raid1_end_read_request will be called.

```
			generic_make_request(read_bio);
```

Now bio is sent to request-queue of vda or vdb.
I checked what function process the bio with gdb.

```
(gdb) p read_bio->bi_bdev->bd_disk->queue->make_request_fn 
$31 = (make_request_fn *) 0xffffffff813a2130 <blk_sq_make_request>
```

blk_sq_make_request will get bio and send it into virtio driver because QEMU generates block disk with virtio block driver.
You can check detail with blk_mq_init_allocated_queue and virtblk_probe functions.

#### raid1_end_read_request

Let's check what raid1 module will do to complete bio processing.

```
	int uptodate = !bio->bi_error;
...
	if (uptodate)
		set_bit(R1BIO_Uptodate, &r1_bio->state);
...
	if (uptodate) {
		raid_end_bio_io(r1_bio);
```

Above code shows that raid1_end_read_request calls raid_end_bio_io if there is no error from vda/vdb device.
Then raid_end_bio_io free r1_bio object.

### WRITE operation

There is the first loop to set bio in bios[].

```
	disks = conf->raid_disks * 2;
 retry_write:
	rcu_read_lock();
	max_sectors = r1_bio->sectors;
	for (i = 0;  i < disks; i++) {
		struct md_rdev *rdev = rcu_dereference(conf->mirrors[i].rdev);
		r1_bio->bios[i] = NULL;
		atomic_inc(&rdev->nr_pending);
		r1_bio->bios[i] = bio;
	}
	rcu_read_unlock();
```

r1_bio object bios[] array to store bios that will be sent to disks, vda/vdb.
We have two raid-disks, so bios array has four elements: two for disks and two for replacement disks.
If we don't have any error on disks, the first loop is the same as ``bios[0] = bios[1] = bio``.
Following shows bios array values.

```
(gdb) p r1_bio->bios[0]
$38 = (struct bio *) 0xffff880005ef4200
(gdb) p r1_bio->bios[1]
$42 = (struct bio *) 0xffff880005ef4200
(gdb) p r1_bio->bios[2]
$43 = (struct bio *) 0x0 <irq_stack_union>
(gdb) p r1_bio->bios[3]
$44 = (struct bio *) 0x0 <irq_stack_union>
```

But the first loop does not anything.
The second loop actually does IO processing.

```
	for (i = 0; i < disks; i++) {
		struct bio *mbio;
		if (!r1_bio->bios[i])
			continue;

		mbio = bio_clone_mddev(bio, GFP_NOIO, mddev);
......
		r1_bio->bios[i] = mbio;

		mbio->bi_iter.bi_sector	= (r1_bio->sector +
				   conf->mirrors[i].rdev->data_offset);
		mbio->bi_bdev = conf->mirrors[i].rdev->bdev;
		mbio->bi_end_io	= raid1_end_write_request;
		mbio->bi_rw =
			WRITE | do_flush_fua | do_sync | do_discard | do_same;
		mbio->bi_private = r1_bio;

		atomic_inc(&r1_bio->remaining);

		cb = blk_check_plugged(raid1_unplug, mddev, sizeof(*plug));
		if (cb)
			plug = container_of(cb, struct raid1_plug_cb, cb);
		else
			plug = NULL;
		spin_lock_irqsave(&conf->device_lock, flags);
		if (plug) {
			bio_list_add(&plug->pending, mbio);
			plug->pending_cnt++;
		} else {
			bio_list_add(&conf->pending_bio_list, mbio);
			conf->pending_count++;
		}
		spin_unlock_irqrestore(&conf->device_lock, flags);
		if (!plug)
			md_wakeup_thread(mddev->thread);
```

It created new bio object for each raid-disks.
And new bio is added into list of plug or list of conf.

If a task is plugged (it means IO is not processed at the moment),
then the bio will be added into list of plug and sent to raid-disks via raid1_unplug function,
that is called by blk_finish_plug.
You can see calling generic_make_request in raid1_unplug.

Or if a task is not plugged (IO can be processed right now),
bio will be added into list of conf and raid1 thread is waken up.
raid1d thread calls flush_pending_writes that calls generic_make_request with bio in conf->pending_bio_list.

Please refer other documents for detail of plug.
* https://lwn.net/Articles/438256/


```
	r1_bio_write_done(r1_bio);
```

r1_bio_write_done is called but r1_bio is not freed until all bio in r1_bio are completed.
The initial value of r1_bio->remaining is 1 and it is added whenever bio is added.
So if there are two bio, r1_bio->remaining value would be three.
r1_bio_write_done decreased r1_bio->remaining so that it is two.
When bio is completed, raid1_end_write_request is called and r1_bio_write_done is called inside of raid1_end_write_request.
So all bio are completed, r1_bio->remaining become zero and r1_bio object is freed.


