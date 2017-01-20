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

### READ operation




### WRITE operation




## md100_raid1 thread (raid1() in raid1.c)

IMPORTANT!! CALL generic_make_request!!!


raid1d: process bio in mddev->private->retry_list


* md_check_recover()




struct ribio

