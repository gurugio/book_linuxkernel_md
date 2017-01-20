# ioctl-STOP_ARRAY

```
/ # mdadm --stop /dev/md100
[   45.075293] md100: detected capacity change from 104726528 to 0
[   45.076719] md: md100 stopped.
[   45.077463] md: unbind<vdc>
[   45.091792] md: export_rdev(vdb)
[   45.092595] md: unbind<vdb>
[   45.103687] md: export_rdev(vda)
mdadm: stopped /dev/md100
```

do_md_stop
* md_unregister_thread(&mddev->thread)
  * stop md100_raid1 thread
* pers->free(mddev, mddev->private)
  * == raid1_free
  * free conf object stored in mddev->private
* set_capacity(disk, 0)
  * set size of gendisk to 0
* export_array(mddev)
  * md_kick_rdev_from_array(rdev)
    * unbind_rdev_from_array(rdev)
      * print "md: unbind<vda>"
    * export_rdev
      * print "md: export_rdev(vda)"
      * kboject_put(&rdev->kobj)
        * rdev->kobj is initialized by ``kobject_init(&rdev->kobj, &rdev_ktype)``
        * kdev_ktype.release = rdev_free
        * kobj_release call kdev_ktype.release and rdev is freed by rdev_free
  * mddev->raid_disks = 0
* md_clean(mddev)
  * remove all information in mddev object
* KOBJ_CHANGE uevent
