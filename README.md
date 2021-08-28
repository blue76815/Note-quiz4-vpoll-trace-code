# quiz3 vpoll 操作方式和程式解析
###### tags: `linux-summer-2021`
<style>
.blue {
  color: blue; 
}
.red {
  color: red;
}
</style>

## 測驗 1 操作方式
step1.先掛載 vpoll

`sudo insmod vpoll.ko`

掛載完可用 `dmesg` 顯示核心訊息
![](https://i.imgur.com/V3ZWiif.png)

`[10056.420070] vpoll: loaded`
![](https://i.imgur.com/L2EPjHh.png)

step2.執行 `./user` 檔
![](https://i.imgur.com/tgF7iTy.png)

step3.使用完卸載 vpoll
`sudo rmmod vpoll`
![](https://i.imgur.com/v0a4iyV.png)

<span class="blue">**注意:若 vpoll 已卸載，則執行`./user`無效**</span> 

```
blue76815@blue76815-virtual-machine:~/桌面/2021成大暑期/quiz4_vpoll/8月25日進度/quiz4-1-type1$ sudo rmmod vpoll 
blue76815@blue76815-virtual-machine:~/桌面/2021成大暑期/quiz4_vpoll/8月25日進度/quiz4-1-type1$ ./user
/dev/vpoll: No such file or directory
blue76815@blue76815-virtual-machine:~/桌面/2021成大暑期/quiz4_vpoll/8月25日進度/quiz4-1-type1$ 
```
![](https://i.imgur.com/MwyWwyq.png)

此step1-step3步驟呼應
**一旦 `vpoll` 掛載進 Linux 核心後，**

**將可藉由 `ioctl` 接受來自使用者層級的命令要求。**(因此老師才設計一個測試碼 `./user` ==>用來呼叫`ioctl` )

### 操作總結
`vpoll.c` :本次設計的考題
`user.c` :測試碼，**用來驗證測試 vpoll 模組功能**用的

<span class="blue">**因此我們先分析主要功能 `vpoll.c` 程式碼原理**</span> 

---

## 1.`vpoll.c` 程式碼分析
Device charactor 的設計方式參閱
* [2021q3 Homework1 (quiz1)](https://hackmd.io/@blue76815/2021q3-linux2021-quiz1)

```
module_init(vpoll_init);//掛載 vpoll 模組，程式進入點為vpoll_init()
module_exit(vpoll_exit);//卸載 vpoll 模組，程式進入點為vpoll_exit()
```

```c
static int __init vpoll_init(void)
{
    int ret;
    struct device *dev;

    if ((ret = alloc_chrdev_region(&major, 0, 1, NAME)) < 0)//申請一個 char device numbers(字元設備號碼)
        return ret;
    vpoll_class = class_create(THIS_MODULE, NAME);//在 sys/class/ 目錄下 創建一個class,
    if (IS_ERR(vpoll_class)) {
        ret = PTR_ERR(vpoll_class);
        goto error_unregister_chrdev_region;
    }
    vpoll_class->devnode = vpoll_devnode;
    dev = device_create(vpoll_class, NULL, major, NULL, NAME);//創建一個設備(在/dev目錄下創建設備文件)，並註冊到sysfs
										//因為我們寫 NAME "vpoll"，所以會創建在 /dev/vpoll 目錄
    if (IS_ERR(dev)) {
        ret = PTR_ERR(dev);
        goto error_class_destroy;
    }
    cdev_init(&vpoll_cdev, &fops);//初始化cdev
    if ((ret = cdev_add(&vpoll_cdev, major, 1)) < 0)//cdev_add()向系統註冊設備
        goto error_device_destroy;

    printk(KERN_INFO NAME ": loaded\n");
    return 0;
	
/*上面註冊過程 若出現 error 則根據 error code 解除註冊功能*/
error_device_destroy:
    device_destroy(vpoll_class, major);
error_class_destroy:
    class_destroy(vpoll_class);
error_unregister_chrdev_region:
    unregister_chrdev_region(major, 1);
    return ret;
}
```
### 1.0. `vpoll_devnode()` 函式介紹
其中
```c
static char *vpoll_devnode(struct device *dev, umode_t *mode)
{
    if (!mode)
        return NULL;

    *mode = 0666;
    return NULL;
}
static struct class *vpoll_class = NULL;
vpoll_class = class_create(THIS_MODULE, NAME);//在 sys/class/ 目錄下 創建一個class,
vpoll_class->devnode = vpoll_devnode;
```
devnode成員來自 [include/linux/device/class.h](https://elixir.bootlin.com/linux/v5.13.12/source/include/linux/device/class.h#L54)

回傳值變數 vpoll_class，也能註冊一組 callback 函式 `char *(*devnode)`

```c=
/**
* @devnode:	Callback to provide the devtmpfs.

* A class is a higher-level view of a device that abstracts out low-level
* implementation details. Drivers may see a SCSI disk or an ATA disk, but,
* at the class level, they are all simply disks. Classes allow user space
* to work with devices based on what they do, rather than how they are
* connected or how they work.
*/
struct class {
	const char		*name;
	struct module		*owner;

	const struct attribute_group	**class_groups;
	const struct attribute_group	**dev_groups;
	struct kobject			*dev_kobj;

	int (*dev_uevent)(struct device *dev, struct kobj_uevent_env *env);
	char *(*devnode)(struct device *dev, umode_t *mode);

	void (*class_release)(struct class *class);
	void (*dev_release)(struct device *dev);

	int (*shutdown_pre)(struct device *dev);

	const struct kobj_ns_type_operations *ns_type;
	const void *(*namespace)(struct device *dev);

	void (*get_ownership)(struct device *dev, kuid_t *uid, kgid_t *gid);

	const struct dev_pm_ops *pm;

	struct subsys_private *p;
};
```

### 1.1. `vpoll.c` 主要功能介紹
`vpoll.c` 主要功能強調在
```c
static const struct file_operations fops = {
    .owner = THIS_MODULE,
    .open = vpoll_open,
    .release = vpoll_release,
    .unlocked_ioctl = vpoll_ioctl,
    .poll = vpoll_poll,
};

static char *vpoll_devnode(struct device *dev, umode_t *mode)
{
    if (!mode)
        return NULL;

    *mode = 0666;
    return NULL;
}

static int __init vpoll_init(void)
{
    ...
    vpoll_class->devnode = vpoll_devnode;
    ...
}
```
### 1.2 file_operations結構體
詳細說明參照
[3.3.3.1. file_operations結構體](http://doc.embedfire.com/linux/imx6/base/zh/latest/linux_driver/character_device.html#file-operations)
> ## 3.3.3.1. file_operations結構體
> <span class="red">**file_operation**</span> 就是把<span class="blue">**系統調用 (system call)**</span> 和<span class="red">**驅動 (Device)程序**</span>關聯起來的<span class="red">**關鍵數據結構**</span>。
> 
> 這個結構的<span class="blue">**每一個成員**</span>都<span class="blue">**對應著一個系統調用 (system call)**</span>。
> 
> <span class="blue">**讀取 file_operation**</span> 中相應的<span class="blue">**函數指標**</span>，接著<span class="blue">**把控制權轉交給函數指標指向的函數**</span>，從而完成了 Linux 設備驅動程序的工作。

根據
```c
static const struct file_operations fops = {
    .owner = THIS_MODULE,
    .open = vpoll_open,
    .release = vpoll_release,
    .unlocked_ioctl = vpoll_ioctl,
    .poll = vpoll_poll,
};
```
我們在 file_operations 只註冊了以下5個函數指標成員

```c
struct file_operations {

    struct module *owner;
    int (*open) (struct inode *, struct file *);
    int (*release) (struct inode *, struct file *);
    __poll_t (*poll) (struct file *, struct poll_table_struct *);
    long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
    
} __randomize_layout;
```
根據 [3.3.3.1. file_operations结构体](http://doc.embedfire.com/linux/imx6/base/zh/latest/linux_driver/character_device.html#file-operations)

* open： 設備驅動第一個被執行的函數，一般用於硬體的初始化。如果該成員被設置為NULL，則表示這個設備的打開操作永遠成功。
* release： 當 file 結構體被釋放時，將會調用該函數。與open函數相反，該函數可以用於釋放
* poll： 用來輪詢 device 是否可以進行 non-blocking 的讀寫
* unlocked_ioctl： 提供設備執行相關控制命令的實現方法，它**對應於應用程式的 `fcntl()` 函數以及 `ioctl()` 函數。在 kernel 3.0 中已經完全刪除了 struct file_operations 中的 ioctl 函數指標。**

因此 `ioctl()` 函數，在 kernel 5.4 已經不存在，此功能已經替換成 `unlocked_ioctl()` 函數

[基礎 Linux Device Driver 驅動程式#9 (IOCTL)](http://csw-dawn.blogspot.com/2012/01/linux-device-driver-9-ioctl.html)

有關 `ioctl()` 如何變遷替換成 `unlocked_ioctl()` 功能的演進，參閱

[**[Linux Kernel慢慢學]Different betweeen ioctl, unlocked_ioctl and compat_ioctl**](https://meetonfriday.com/posts/736969d7/)
> ## ioctl是什麼?
> `ioctl()` 是撰寫driver一個很重要的接口，以字元裝置驅動(char device driver)來說，**透過這個接口可以讓user來操作driver執行一些行為。**
> 
> 在撰寫 driver code 時，我們必須透過 `register_chrdev()`來向kernel註冊我們的 driver 。為此我們需要**提供該 driver 的 file_operation 相關函數實作**來**讓 user 可以透過這些接口來操控該 driver。**
> 
> ## Big Kernel Lock下的舊產物: ioctl
> **ioctl()** 在2.6版本以前是還有這個 function 的，例如你可以在[2.5.75的fs.h](https://elixir.bootlin.com/linux/v2.5.75/source/include/linux/fs.h#L718)中看到。但**在2.6以後就被替換成 `unlocked_ioctl()` 了**，為什麼呢?
> 由於 `ioctl()` 是早期仍然在 **BKL(Big Kernel Lock)** 機制下執行的產物(BKL是甚麼可以再去google，是kernel發展史中蠻有趣的一段歷史)，**BKL的機制會使得`ioctl()`的運作時間較長，可能會造成其他process的延遲。**
> 
> 隨著 Kernel 後續的改版，BKL不再是一個需要的機制了，大家開始把被 BKL 保護的 function 移除 BKL。但一下子就把 BKL 完全移除還是會有顧慮，大家應該更加仔細的去審視是否有需要自己加入新的lock來保護自己的程序。**所以此時需要一個過渡的機制: `unlocked_ioctl()`**
> 
> 如果某個驅動的fops提供了 `unlocked_ioctl()`，那麼他將優先調用 `unlocked_ioctl()` 而不是有BKL版本的`ioctl()`。
> 
> * **`unlocked_ioctl() `不再提供inode參數，但你仍可以透過`filp->f_dentry->d_inode` 來取得**
> * **`unlocked_ioctl()`不再使用BKL，工程師需要根據自己的需求來決定要不要加入lock的機制**
> 
> 所以我們知道了**原始 `unlocked_ioctl()` 的誕生是為了應付一段過渡期，讓大家能夠在這段時間趕快去修改自己的`ioctl()`**，例如你在[2.6.11版本的fs.h](https://elixir.bootlin.com/linux/v2.6.11/source/include/linux/fs.h#L931)就可以看到這兩個接口是同時存在的。
> 
> * **而在2.6.36後就正式將ioctl()移除了**，大家都必須透過 `unlocked_ioctl()` 來提供ioctl的接口
> 
> 不過這就只是一段歷史，**對於現在的driver開發者也沒什麼影響，就是把自己寫好的 `ioctl` 接到 `unlocked_ioctl()` 上面去而已。**
> 
> ## 為了相容性而出現的compat_ioctl
> 在 Michael s. Tsirkin 發布的 patch 提供了`unlocked_ioctl`的同時也提供了另外一個接口: `compat_ioctl()`。
> > If this method exists, it will be called (without the BKL) whenever a 32-bit process calls ioctl() on a 64-bit system. It should then do whatever is required to convert the argument to native data types and carry out the request
> 
> 他出現的目的很簡單，就是相容性: **為了讓32-bit的process可以在64-bit上的system來執行`ioctl()`(沒有BKL版本)。**

對應到 成大 The Linux Kernel Module Programming Guide
[9 Talking To Device Files](https://sysprog21.github.io/lkmpg/#talking-to-device-files)
<span class="blue">裡面還在介紹過時的 `ioctl()`</span>，我得 push request 更新
![](https://i.imgur.com/dbCkT66.png)

### 1.3 在 user space 呼叫 `ioctl()` 函數
比較 `driver` 端 ( `module.c` 驅動模組)和 user space端( `user.c` 應用程式)
有關 `ioctl()` 的設定


#### module.c :為 driver 端驅動模組，io control 註冊到 **`.unlocked_ioctl`**
```c
static const struct file_operations fops = {
    ...
    .unlocked_ioctl = vpoll_ioctl,
    ...
};

#define NAME "vpoll"
static int __init vpoll_init(void)
{
    ....
    dev = device_create(vpoll_class, NULL, major, NULL, NAME);//創建一個設備(在/dev目錄下創建設備文件)，並註冊到sysfs
                                                            //因為我們寫 NAME "vpoll"，所以會創建在 /dev/vpoll 目錄
    ....
}
```

#### user.c :為 user space 應用程式，用 [ioctl()](https://man7.org/linux/man-pages/man2/ioctl.2.html) 指令去呼叫 vpoll 檔案描述符的 io 資料

```c
int main(int argc, char *argv[])
{
    ...
    int efd = open("/dev/vpoll", O_RDWR | O_CLOEXEC);
    ...
    switch (fork()) {
    case 0://在子行程
        sleep(1);
        ioctl(efd, VPOLL_IO_ADDEVENTS, EPOLLIN);
        sleep(1);
        ioctl(efd, VPOLL_IO_ADDEVENTS, EPOLLIN);
        sleep(1);
        ioctl(efd, VPOLL_IO_ADDEVENTS, EPOLLIN | EPOLLPRI);
        sleep(1);
        ioctl(efd, VPOLL_IO_ADDEVENTS, EPOLLPRI);
        sleep(1);
        ioctl(efd, VPOLL_IO_ADDEVENTS, EPOLLOUT);
        sleep(1);
        ioctl(efd, VPOLL_IO_ADDEVENTS, EPOLLHUP);
        exit(EXIT_SUCCESS);
    ...
}
```
ioctl() 從 user space 如何控制到 driver模組的過程 
參閱
[**ioctl()分析——從用戶空間到設備驅動**](https://www.twblogs.net/a/5bd39e952b717778ac2098d4)
> 一個字符設備驅動通常會實現常規的打開、關閉、讀、寫等功能，但在一些細分的情境下，如果需要擴展新的功能，通常以增設ioctl()命令的方式實現。
> ![](https://i.imgur.com/i0RxGf5.png)
> 
> ## 用戶空間(user space)的 [ioctl()](https://man7.org/linux/man-pages/man2/ioctl.2.html)
> ```
> #include <sys/ioctl.h> 
> int ioctl(int fd, int cmd, ...) ;
> ```
> 
> ## 驅動中的 ioctl()
> 設計driver模組時(module.c)，對應註冊的函數指標成員為
> ```c
> struct file_operations {
> 
>     long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
>     long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
>     
> } __randomize_layout;
> ```
> * 在新版內核中，**`unlocked_ioctl()` 與 `compat_ioctl()` 取代了 `ioctl()`<span class="blue">(此名稱是指在舊版kernel內的 file_operations函數指標成員名稱已經不存在，但是在user space中是用 ioctl()呼叫driver模組中的IO資料)</span>**
> * unlocked_ioctl(): 在無大內核鎖（BKL）的情況下調用
> * compat_ioctl(): **為了讓32-bit的process可以在64-bit上的system來執行ioctl()(沒有BKL版本)。**

在 Linux 官方文件的 [ioctl](https://linux-kernel-labs.github.io/refs/heads/master/labs/device_drivers.html?highlight=ioctl#ioctl) 介紹
> ## ioctl
> In addition to read and write operations, a driver needs the ability to perform certain physical device control tasks. 
> 
> These operations are accomplished by implementing a **ioctl function**. 
> 
> Initially, the **ioctl system call** used **Big Kernel Lock**. That's why the call was gradually **replaced with its unlocked version called `unlocked_ioctl`**. 
> You can read more on LWN: http://lwn.net/Articles/119652/

---

在 http://lwn.net/Articles/119652/ 連結內的文章
即LWN.net報導 The new way of ioctl()
裡面的介紹內容就是 
[**[Linux Kernel慢慢學]Different betweeen ioctl, unlocked_ioctl and compat_ioctl**](https://meetonfriday.com/posts/736969d7/) 中提到  file_operation 內的 `unlocked_ioctl()` 與 `compat_ioctl()` 取代了 `ioctl()`的緣由

點旁邊的 Big kernel lock 索引
![](https://i.imgur.com/RKUZxPe.png)

![](https://i.imgur.com/iEoBKsJ.png)
找到
[Linux support for ARM big.LITTLE](https://lwn.net/Articles/481055/)
即老師書上提到的 ARM big.LITTLE 
### [Waiting queues](https://linux-kernel-labs.github.io/refs/heads/master/labs/device_drivers.html?highlight=ioctl#waiting-queues)
## 2.`user.c` 程式碼分析 




---

