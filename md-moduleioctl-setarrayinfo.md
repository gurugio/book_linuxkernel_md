# ioctl-SET_ARRAY_INFO



```
static int md_alloc(dev_t dev, char *name)
{
......
	disk->private_data = mddev;
```




```
static int md_ioctl(struct block_device *bdev, fmode_t mode,
			unsigned int cmd, unsigned long arg)
{
......
	mddev = bdev->bd_disk->private_data;

```