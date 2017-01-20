# io handling of md module

md_make_request -> mddev->pers->make_request = raid1.c:make_request

## raid1.c:make_request

```
	r1_bio = mempool_alloc(conf->r1bio_pool, GFP_NOIO);
```

VERY VERY IMPORTANT!!




## raid1d
raid1d: process bio in mddev->private->retry_list
