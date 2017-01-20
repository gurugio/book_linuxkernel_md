# io handling of md module

bio handling of md driver is done by ``md_make_request()``.

md_alloc():
```
	blk_queue_make_request(mddev->queue, md_make_request);
```

md_make_request -> mddev->pers->make_request = raid1.c:make_request

## raid1.c:make_request

```
	r1_bio = mempool_alloc(conf->r1bio_pool, GFP_NOIO);
```

VERY VERY IMPORTANT!!






## md100_raid1 thread (raid1() in raid1.c)

IMPORTANT!! CALL generic_make_request!!!


raid1d: process bio in mddev->private->retry_list


* md_check_recover()




struct ribio

