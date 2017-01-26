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




## md100_raid1 thread (raid1() in raid1.c)

IMPORTANT!! CALL generic_make_request!!!


raid1d: process bio in mddev->private->retry_list


* md_check_recover()




struct ribio

