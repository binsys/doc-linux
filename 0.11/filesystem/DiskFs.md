Linux 0.11 安装硬盘文件系统
================================================================================

概述
--------------------------------------------------------------------------------

### MINIX文件系统:

MINIX文件系统不过是在用户空间中运行的一个大型C程序,读写文件时,用户进程向文件系统发送一条消息,文件系统进行相应处理后,返回结果.
实际上,MINIX文件系统可以看成是和调用进程在同一台主机上运行的网络文件服务程序.
文件系统的结构基本上与内存管理器和所有的I/O任务一样.文件系统的主循环程序不断地等待消息.当收到消息后,它首先提取消息的类型,
然后以其为索引查找文件系统中处理各类消息的过程指针表.随后调用相应过程,进行处理后,返回状态值.文件系统再把回答消息发送给调用进程,
然后回到循环的开始,等待下一条消息的到来。

MINIX文件系统是一个逻辑的, 自包含的实体. 它含有i-节点, 目录和数据块。MINIX文件系统可以存储在任何块设备中,例如软盘或一个硬盘分区。
MINIX的文件系统都有相同的布局. 如下所示是一个128个i-节点和1K块的360K软盘的布局.更大的系统,或者是那些有不同的i-节点数和块大小的系统,
也同样由这6部分顺序组成,但是各部分之间的相对大小可能不一样:

```
{引导块(1)}{超级块(1)}{i-节点位图块(1)}{区段位图(1)}{i节点(n)}{数据块(n)}
```

#### 引导块:

每个文件系统都以引导块开始,引导块中包含有可执行代码.启动计算机时,硬件从引导设备将引导块读入内存,
转而执行其代码.引导块代码开始操作系统本身的加载过程。一旦系统启动之后,引导块不再使用.并非每个
磁盘驱动器均可用作引导设备,但是为了保持结构的一致,每个块设备都为引导块代码保留一块.这种方法最多
不过浪费了一个块. 为防止硬件从非启动设备上启动,在将可执行代码写入引导设备的引导块中时,在引导块的已
知位置处写入魔数。从一个设备上启动时,硬件(实际上是BIOS代码)首先检测魔数是否存在。若不存在,则拒绝把
引导块载入内存,这样可以防止把垃圾用作引导程序。

#### 超级块:

超级块(super-block)中含有文件系统的布局信息.它的主要功能是: 给出文件系统不同部分的大小.如果给定块大小和i-节点数,
我们很容易算出i-节点位图的大小和存放i-节点所需的块数。例如:

```
1K的块,每个位图块有1K字节(8K位),可以记录8192个i-节点的状态(实际上第一块只能处理8191个i-节点,因为0号i-节点并不存在,
但我们在位图中也为它保留一位).10000个i-节点, 要用到两个位图块.每个i-节点占64字节,1K的块中可以有16个i-节点。
如果有128个可用的i-节点,则需要8个磁盘块来存放。
```

磁盘存储区可以以区段为单位进行分配,而每个区段可以包含1,2,4,8个,或一般情况下,2n个磁盘块.区段位图按区段,
而不是块来记录空闲存储区。MINIX用到的所有标准软盘中,区段大小和块大小是一样的(均为1K),因此在这些设备上,
可以近似把区段看成是块.

可以注意到: 每个区段包含的块数并没有存放在超级块中,我们并不需要这一数据.我们存放的是底为2, 区段数除以块数
所得值的对数. 根据它,可以知道从区段转换成块或者从块转换成区段要移位的次数. 例如:

```
每个区段中含有8个块,log2^8=3,因此要找到包含第128块的区段,我们可以把128右移3位,得到16.
```

区段位图中只包含数据区段(即位图和i-节点用到的块并不在该位图中).第一个数据区段在位图中用区段1表示,
同i-节点位图一样,区段位图中第0位也未使用,因此第一个区段位图块只能映射到8191个区段,以后的每块可以映射到8192个区段.
考察一下新格式化磁盘的位图,可以发现i-节点和区段位图中均有2位为1.一位是不存在的0号i-节点和0号区段,
而另一位是设备根目录使用的i-节点和区段,在文件系统创建时,根目录自动存在.

我们还可以注意到: 超级块中的信息冗余. 由于我们有1K空间可用于存放超级块的信息,因此我们可以事先按不同的形式算出所需的信息,
而不必在使用时重新计算. 例如,磁盘上的第一个数据区段的区段号,可以从块大小,区段大小,i-节点数以及区段数计算得到. 但是,
直接把它存放在超级块中要方便得多. 超级块中的剩下部分总归是要浪费的,我们还不如用它来存储一些有用的数据。

在MINIX启动时,根设备中的超级块被读入内存中,同样,安装其他文件系统时,他们的超级块也读入内存。内存的超级块表
中有些域不出现在磁盘上, 其中包括指定设备打开方式和字节顺序的标志等,此外,内存的超级块表还含有指向位图第一个空闲位的域,
使用这个域可提高访问速度,以及表征超级块所属设备的一个域。实用程序mkfs可用来创建文件系统。我们可以通过像:

```
mkfs /dev/fd1 1440
```

的命令行调用该程序,在驱动器1中的软盘上创建1440个块的空文件系统,这一命令还在超级块中写入魔数,表明该文件系统是一个
有效的MINIX文件系统。此外,我们也可以通过一个原型文件来调用mkfs,其中列出要包含在新文件系统中的目录和文件。MINIX文件
系统经过改进,有些方面(比如i-节点大小)与早期版本不同,魔数还可以表明创建文件系统的mkfs的版本,从而处理他们之间的不同.
MOUNT系统调用检查超级块中魔数和其他信息,可以拒绝安装不是MINIX格式的文件系统,

#### 位图

MINIX用两个位图来记录空闲i-节点和空闲区段。当文件被删除时,很容易算出哪一个位图块包含了所释放i-节点的相应位,
利用通常的高速缓存机制查找该块,一旦找到,相应于被释放i-节点的那一位清0。区段在区段位图中的释放也类似。

逻辑上,在创建文件时,文件系统必须在位图块中查找第一个空闲i-节点,把它分配给这个新创建的文件. 然而, 超级块在内存的拷贝中
有一个域指向第一个空闲i-节点, 因此不必进行查找, 在该空闲i-节点分配使用后,就需要修改指针,使它指向下一个空闲i-节点,
往往是下一个或者较近的一个节点。同样地, i-节点被释放后,检查这个i-节点是否位于第一个空闲i-节点的前面,若是,
则需要修改指向第一个空闲i-节点的指针. 如果磁盘上的所有i-节点全被使用,查找函数返回0,这也是0号i-节点未使用的原因
(即它可以用于表明查找失败)(在mkfs创建新文件时,它把0号i-节点清零,并把位图中的最低位设置为1,防止文件系统分配0号i-节点).
上面所讲的同样也适用于区段位图。 逻辑上,申请空间时,需要在区段位图中查找第一个空闲区段,但是超级块的内存拷贝保存了指向
第一个空闲区段的指针,因而消除了很多顺序查找位图的麻烦。

**区段和块之间的不同**:

```
使用区段的目的是: 确保同一文件的所有磁盘块都位于同一个柱面上,从而改进顺序读取文件的性能.我们采用了一次分配多个块的方法.
假设块大小为1K,而区段大小为4K,区段位图中记录了区段,而非块的使用情况。一个20M的磁盘有5K个4K大小的区段,因此其区段位图需要5K位。
文件系统的大多数部分都以块为单位进行操作的.磁盘每次传送一块,高速缓存也按照块进行处理,系统中只有记录物理磁盘地址的一小部分
(例如区段位图和i-节点)需要知道区段的存在.
```

**历史**:

A. 1985年,当MINIX还处于构思阶段的时候,磁盘容量很小,许多用户只有软盘。在V1文件系统中,我们决定把磁盘地址限制在16位,
这样能够把大多数地址存放在间接块中。16位的区段号和1K大小的区段只能寻址64K个区段,从而将磁盘容量限制为64M.
那时,这可是一个相当大的容量,我们当时还考虑到,需要扩充磁盘容量时,很容易把区段大小转换为2K或者4K,而不必改动块的
大小。16位的区段号还容易使得i-节点的大小保持为32字节。

B. 由于MINIX不断发展,同时大磁盘更加普及,因此有必要对MINIX的文件系统进行修改。许多文件长度小于1K,
因此增加块大小意味着浪费磁盘带宽和读写几乎是空的块,也意味着把它存放在高速缓存时浪费了宝贵的主存。
原本我们是可以增加区段大小的,但是大的区段浪费了更多的磁盘空间,何况我们还希望保持对小磁盘操作的高效率。
当然,另一种合理的解决方案是在不同大小的设备中使用不同大小的区段。但最终我们决定把磁盘指针的大小增加到32位。
这使得在块大小和区段大小均为1K时,MINIX V2版本的文件系统可以处理容量达4TB的设备。作出这种决定的另外一个原因与
i-节点的更改有关.由于i-节点中包含的内容增加了,为此我们把i-节点的大小也增加到64个字节。

区段的使用也带来了意料不到的问题,我们用一个简单的例子来阐述:

```
考虑4K的区段和1K的块,假设文件长度为1K,这时文件被分配了一个区段,1K和4K之间的磁盘块中含有垃圾(以前用户的残留数据),
但是这不会对我们造成不利。因为在i-节点中文件大小清楚地标记为1K.事实上,包含垃圾的磁盘块根本不会读到高速缓存中,
因为读操作是以块,而不是区段为单位进行的.超出文件尾的读操作总是返回0,不包含任何数据.
假设现在有人将文件指针定在32768字节并写入1个字节数据,文件长度变为32769字节.随后读取1K后面的数据将返回该块以前的内容,
从而形成一个很大的安全缺口.解决这一问题的方法是,执行写操作时,检查写入位置是否超出了文件尾.若超出,则将文件的最后一个
区段中所有还未分配的块清空. 尽管这种情况很少发生,然而我们的代码必须进行处理,这使得MINIX文件系统更加复杂。
```

#### i-节点

MINIX中i-节点的布局几乎与标准UNIX的i-节点相同。磁盘区段指针是一些32位的指针,总共有9个这样的指针: 7个直接的,2个间接的。
MINIX的i-节点占64个字节,这也同标准UNIX的i-节点一样。有一个未使用的空间可以用于第10个(三次间接)指针,我们在MINIX文件系统
的标准版本中还不支持这一指针。MINIX i-节点中的存取时间,修改时间以及i-节点的修改时间都和标准UNIX一样。i-节点的修改时间在
大多数文件操作时都要进行修改,只有读文件除外。

在打开文件时,首先要找到文件的i-节点,并把它载入内存的inode表中,直至关闭前,它一直保存在内存中。内存的inode表中有一些域不出现在磁盘上,
例如, i-节点所在设备的设备号以及在该设备上的i-节点号,通过这两个值文件系统可以知道在内存中数据修改后,要将这些数据写到何处。每个i-节点
还有一个计数器,当文件多次打开时,在内存中只保存一个i-节点拷贝,但是每次打开该文件,计数器加1,每次关闭该文件,计数器减1。只有在计数器减到0时,
才将i-节点从inode表中删除。 若i-节点自上次调入内存之后被修改过,则要将它写回磁盘。

文件i-节点的主要功能是: 给出文件数据块所在的位置。前7个区段号就放在i-节点结构之中。对于MINIX标准发行版,区段大小和块大小均为1K,
因此小于7K的文件不必使用间接块。 如果文件长度超过7K,就要使用间接区段。MINIX中它只用到了一次间接块和两次间接块。若块大小和区段
大小均为1K,区段号为32位,则一次间接块含有256项,可以表示1/4M的存储区,两次间接块指向256个一次间接块,因此可以存取长达64M的文件。
MINIX文件系统中文件的最大长度为1G,我们可以使用三次间接块或修改区段大小来存取大于64M的文件。

i-节点中还包含有模式信息, 它给出了文件的类型(正规文件,目录,块设备文件,字符设备文件或管道)以及保护标志、SETUID位和SETGID位。
i-节点的链接数目域记录了有多少个目录项指向这个i-节点,因此文件系统知道什么时候该释放文件的存储区。我们不应当把它与打开文件计数器
(只出现在内存的inode表中,不在磁盘上)相混淆,后者指出了文件被打开的次数,而且往往是被不同的进程所打开。

准备过程
--------------------------------------------------------------------------------

### 轮转到进程1执行

执行进程1过程如下:

path: init/main.c
```
void main(void)
{
    ......
    if (!fork()) {        /* we count on this going ok */
        init();  // 跳转到进程1执行
    }
    ......
}
```

path: init/main.c
```
void init(void)
{
    ......
    setup((void *) &drive_info); // drive_info就是硬盘参数表
    ......
}
```

### 根据机器系统数据(DRIVE_INFO)设置硬盘参数(hd_info):

#### 硬盘参数表(DRIVE_INFO):

硬盘参数表是由setup程序利用ROM BIOS的中断读取的机器系统数据.

path: init/main.c
```
#define DRIVE_INFO (*(struct drive_info*)0x90080)  内存0x90080开始32字节的机器系统数据用于存放硬盘参数表.
```

#### 硬盘参数描述符:

所有硬盘的参数存放到类型为(struct hd_i_struct)的变量hd_info中

path: kernel/blk_drv/hd.c
```
struct hd_i_struct {
    int head; // 磁头数
    int sect; // 每磁道扇区数
    int cyl;  // 柱面数
    int wpcom;
    int lzone;
    int ctl;
};
```

#### 硬盘信息描述符:

所有的硬盘信息存放到类型为(struct hd_struct)的变量hd中

path: kernel/blk_drv/hd.c
```
static struct hd_struct {
    long start_sect;  // 起始扇区
    long nr_sects;    // 总扇区数
} hd[5*MAX_HD]={{0,0},};
```

#### 过程:

设置硬盘参数信息是通过调用sys_setup来实现的,具体过程如下:

根据硬盘参数表中的信息设置hd_info; 根据hd_info中的硬盘参数描述符中的信息设置hd

具体实现如下所示:

path: kernel/blk_drv/hd.c
```
/* This may be used only once, enforced by 'static int callable' */
int sys_setup(void * BIOS)
{
    static int callable = 1;
    int i,drive;
    unsigned char cmos_disks;
    struct partition *p;
    struct buffer_head * bh;

    if (!callable)
        return -1;
    callable = 0;
#ifndef HD_TYPE
    /* A. 读取drive_info设置hd_info */
    for (drive=0 ; drive<2 ; drive++) {
        hd_info[drive].cyl = *(unsigned short *) BIOS;
        hd_info[drive].head = *(unsigned char *) (2+BIOS);
        hd_info[drive].wpcom = *(unsigned short *) (5+BIOS);
        hd_info[drive].ctl = *(unsigned char *) (8+BIOS);
        hd_info[drive].lzone = *(unsigned short *) (12+BIOS);
        hd_info[drive].sect = *(unsigned char *) (14+BIOS);
        BIOS += 16;
    }
    /* 判断有几个硬盘 */
    if (hd_info[1].cyl)
        NR_HD=2;
    else
        NR_HD=1;
#endif
    /* 一个物理硬盘最多可以分为4各逻辑盘,0是物理盘,1~4是逻辑盘,共5个.
     * 第1个物理盘是0 * 5, 第2个物理盘是1 * 5.
     */
    for (i=0 ; i<NR_HD ; i++) {
        hd[i*5].start_sect = 0;
        hd[i*5].nr_sects = hd_info[i].head*
                hd_info[i].sect*hd_info[i].cyl;
    }

    /*
        We querry CMOS about hard disks : it could be that
        we have a SCSI/ESDI/etc controller that is BIOS
        compatable with ST-506, and thus showing up in our
        BIOS table, but not register compatable, and therefore
        not present in CMOS.

        Furthurmore, we will assume that our ST-506 drives
        <if any> are the primary drives in the system, and
        the ones reflected as drive 1 or 2.

        The first drive is stored in the high nibble of CMOS
        byte 0x12, the second in the low nibble.  This will be
        either a 4 bit drive type or 0xf indicating use byte 0x19
        for an 8 bit type, drive 1, 0x1a for drive 2 in CMOS.

        Needless to say, a non-zero value means we have
        an AT controller hard disk for that drive.
    */

    if ((cmos_disks = CMOS_READ(0x12)) & 0xf0)
        if (cmos_disks & 0x0f)
            NR_HD = 2;
        else
            NR_HD = 1;
    else
        NR_HD = 0;
    for (i = NR_HD ; i < 2 ; i++) {
        hd[i*5].start_sect = 0;
        hd[i*5].nr_sects = 0;
    }

    /* 第一个物理盘号是0x300, 第2个是0x305, 读每个物理硬盘的0号块，即引导块, 有分区信息 */
    for (drive=0 ; drive<NR_HD ; drive++) {
        if (!(bh = bread(0x300 + drive*5,0))) {
            printk("Unable to read partition table of drive %d\n\r",
                drive);
            panic("");
        }
        if (bh->b_data[510] != 0x55 || (unsigned char)
            bh->b_data[511] != 0xAA) {
            printk("Bad partition table on drive %d\n\r",drive);
            panic("");
        }
        p = 0x1BE + (void *)bh->b_data;
        for (i=1;i<5;i++,p++) {
            hd[i+5*drive].start_sect = p->start_sect;
            hd[i+5*drive].nr_sects = p->nr_sects;
        }
        brelse(bh);
    }
    ......
}
```

### 读取硬盘引导块.

#### 过程:

读取硬盘引导块是通过在sys_setup中通过向bread函数传递硬盘设备号和块号0(引导块)来读取的,具体过程如下:

path: kernel/blk_drv/hd.c
```
/* This may be used only once, enforced by 'static int callable' */
int sys_setup(void * BIOS)
{
    ......
    struct partition *p;
    struct buffer_head * bh;
    ......
    for (drive=0 ; drive<NR_HD ; drive++) {
        /* 第一个物理盘号是0x300, 第2个是0x305, 读每个物理硬盘的0号块，即引导块, 有分区信息 */
        if (!(bh = bread(0x300 + drive*5,0))) {
            printk("Unable to read partition table of drive %d\n\r",
                drive);
            panic("");
        }
        if (bh->b_data[510] != 0x55 || (unsigned char)
            bh->b_data[511] != 0xAA) {
            printk("Bad partition table on drive %d\n\r",drive);
            panic("");
        }
        p = 0x1BE + (void *)bh->b_data;
        for (i=1;i<5;i++,p++) {
            hd[i+5*drive].start_sect = p->start_sect;
            hd[i+5*drive].nr_sects = p->nr_sects;
        }
        brelse(bh);
    }
    if (NR_HD)
        printk("Partition table%s ok.\n\r",(NR_HD>1)?"s":"");

    rd_load();
    mount_root();
    return (0);
}
```

进入bread函数中以后调用getblk在缓冲区中申请一个空闲的缓冲块(struct buffer_head):

path: fs/buffer.c
```
/*
 * bread() reads a specified block and returns the buffer that contains
 * it. It returns NULL if the block was unreadable.
 */
struct buffer_head * bread(int dev,int block) // 读制定dev, block, 第一块硬盘是0x300, block是0
{
    struct buffer_head * bh;

    if (!(bh=getblk(dev,block)))
        panic("bread: getblk returned NULL\n");
    if (bh->b_uptodate)
        return bh;
    ll_rw_block(READ,bh);
    wait_on_buffer(bh);
    if (bh->b_uptodate)
        return bh;
    brelse(bh);
    return NULL;
}
```

进入getblk后首先调用get_hash_table查找hash表，检索此前是否有程序把现在要读的硬盘逻辑块
(相同设备号和块号)已经读到缓冲区. 使用hash表的目的是提高查询速度:

path: fs/buffer.c
```
struct buffer_head * getblk(int dev,int block)
{
    struct buffer_head * tmp, * bh;

repeat:
    if ((bh = get_hash_table(dev,block)))
        return bh;
    tmp = free_list;
    do {
        if (tmp->b_count)
            continue;
        if (!bh || BADNESS(tmp)<BADNESS(bh)) {
            bh = tmp;
            if (!BADNESS(tmp))
                break;
        }
    /* and repeat until we find something good */
    } while ((tmp = tmp->b_next_free) != free_list);
    if (!bh) {
        sleep_on(&buffer_wait);
        goto repeat;
    }
    wait_on_buffer(bh);
    if (bh->b_count)
        goto repeat;

    while (bh->b_dirt) {
        sync_dev(bh->b_dev);
        wait_on_buffer(bh);
        if (bh->b_count)
            goto repeat;
    }

    /* NOTE!! While we slept waiting for this block, somebody else might */
    /* already have added "this" block to the cache. check it */
    if (find_buffer(dev,block))
        goto repeat;
    /* OK, FINALLY we know that this buffer is the only one of it's kind, */
    /* and that it's unused (b_count=0), unlocked (b_lock=0), and clean */
    bh->b_count=1;
    bh->b_dirt=0;
    bh->b_uptodate=0;
    remove_from_queues(bh);
    bh->b_dev=dev;
    bh->b_blocknr=block;
    insert_into_queues(bh);
    return bh;
}
```

进入get_hash_table后调用find_buffer从hash表查找缓冲区中是否有指定设备号,块号的缓冲块,
如果能找到指定缓冲块，就直接用:

path: fs/buffer.c
```
/*
 * Why like this, I hear you say... The reason is race-conditions.
 * As we don't lock buffers (unless we are readint them, that is),
 * something might happen to it while we sleep (ie a read-error
 * will force it bad). This shouldn't really happen currently, but
 * the code is ready.
 */
struct buffer_head * get_hash_table(int dev, int block)
{
    struct buffer_head * bh;

    for (;;) {
        if (!(bh=find_buffer(dev,block)))
            return NULL;
        bh->b_count++;
        wait_on_buffer(bh);
        if (bh->b_dev == dev && bh->b_blocknr == block)
            return bh;
        bh->b_count--;
    }
}
```

#### 块高速缓存(struct buffer_head)

块高速缓存用来改进文件系统性能. 高速缓存用一个缓冲数组来实现,其中每个缓冲区由包含指针、计数器和标志的头以及用于存放磁盘块的体组成。
所有未使用的缓冲区均使用双链表, 按最近一次使用时间从近到远的顺序链接起来.

为了迅速判断某一块是否在内存中,我们使用了哈希表。所有缓冲区,如果它所包含块的哈希代码为k,在哈希表中用第k项指向的单链表链接在一起。
哈希函数提取块号低n位作为哈希代码,因此不同设备的块可以出现在同一哈希链之中.每个缓冲区都在其中某个链中。在Linux 0.11启动,初始化文件系统时,
所有缓冲区均未使用,并且全部在哈希表第0项指向的单链表中. 这时,哈希表其他项均为空指针.但是一旦系统启动完成,缓冲区将从0号链中删除,
放到其他链中。

### 从引导块中获取信息