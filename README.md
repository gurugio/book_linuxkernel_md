# md driver for RAID device (On translation from Korea into English)

https://gurugio.gitbooks.io/book_linuxkernel_md/content/

We've done reading block layer and ramdisk driver, next is md dirver that is very common and popular for server market.

To understand md (multi-device) driver, we first know how to use mdadm tool that controls behavior of md driver.
So let's first read source code of mdadm tool and check what command it send to md driver.
And then we will read md driver sources in kernel directory.

_PS. I welcome any help for translation._

INDEX

* [mdadm\/introduction](mdadmintroduction.md)
* [mdadm\/main](mdadmmain.md)
* [mdadm\/create](mdadmcreate.md)
* [mdadm\/create\_2nd](mdadmcreate2nd.md)
* [mdadm\/sequence\_syscall](mdadmsequencesyscall.md)
* [mdadm\/build\_static](mdadmbuildstatic.md)
* [md-module\/md\_init](md-modulemdinit.md)
* [md-module\/md\_alloc](md-modulemdalloc.md)
* [md-module\/ioctl-RAID\_VERSION](md-moduleioctl-raidversion.md)
* [md-module\/ioctl-GET\_ARRAY\_INFO](md-moduleioctl-getarrayinfo.md)
* [md-module\/ioctl-SET\_ARRAY\_INFO](md-moduleioctl-setarrayinfo.md)
* [md-module\/ioctl-ADD\_NEW\_DISK](md-moduleioctl-addnewdisk.md)
* [md-module\/ioctl-RUN\_ARRAY](md-moduleioctl-runarray.md)
* [md-module\/ioctl-STOP\_ARRAY](md-moduleioctl-stoprray.md)
* [md-module\/io](md-moduleio.md)
