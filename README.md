# md driver for RAID device (On translation from Korea into English)

https://gurugio.gitbooks.io/book_linuxkernel_md/content/

We've done reading block layer and ramdisk driver, next is md dirver that is very common and popular for server market.

To understand md (multi-device) driver, we first know how to use mdadm tool that controls behavior of md driver.
So let's first read source code of mdadm tool and check what command it send to md driver.
And then we will read md driver sources in kernel directory.

_PS. I welcome any help for translation._
