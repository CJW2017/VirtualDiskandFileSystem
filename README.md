## 虚拟磁盘和FAT文件系统

1. 通过vhd文件模拟机械磁盘，以FAT16方式分区，实现磁盘文件的复制、读取、删除等操作。
2. 建立虚拟磁盘的缓冲文件系统，实现文件打开关闭、读写、定位等操作函数。

### 磁盘原理

##### 磁盘的逻辑结构

硬盘由很多盘片(platter)组成，每个盘片的每个面都有一个读写磁头。每个盘片被划分成若干个同心圆磁道，每个盘片的划分规则通常是一样的。这样每个盘片的半径均为固定值R的同心圆再逻辑上形成了一个以电机主轴为轴的柱面(Cylinders)，从外至里编号，每个盘片上的每个磁道又被划分为几十个扇区(Sector)，通常的容量是512byte，并按照一定规则编号为1、2、3……形成 Cylinders×Heads×Sector个扇区。

##### MBR(master boot record)扇区：

位于整个硬盘的0柱面0磁头1扇区，bios在执行自己固有的程序以后就会jump到mbr中的第一条指令。将系统的控制权交由mbr来执行。在总共512byte的主引导记录中，MBR的引导程序占了其中的前446个字节(偏移0H~偏移1BDH)，随后的64 个字节(偏移1BEH~偏移1FDH)为DPT(Disk PartitionTable，硬盘分区表)，最后的两个字节“55 AA”(偏移1FEH~偏移1FFH)是分区有效结束标志。

##### 硬盘分区表

操作系统为了便于用户对磁盘的管理。加入了磁盘分区的概 念。即将一块磁盘逻辑划分为几块。在DPT共64个字节中，以16个字节为分区表项单位描述一个分区的属性。

##### DBR区

DBR区(DOS BOOT RECORD)即操作系统引导记录区的意思，通常占用分区的第0扇区共512个字节(特殊情况也要占用其它保留扇区，我们先说第0扇)。在这512个字节中，其实又是由跳转指令，厂商标志和操作系统版本号，BPB(BIOS Parameter Block)，扩展BPB，os引导程序，结束标志几部分组成。

##### FAT表

FAT表(File Allocation Table 文件分配表)，是Microsoft在FAT文件系统中用于磁盘数据(文件)索引和定位引进的一种链式结构。FAT16表实际上是一个数据表，以2个字节为单位，通常情况其第1、2个记录项(前4个字节)用作介质描述。从第三个记录项开始记录除根目录外的其他文件及文件夹的簇链情况。根据簇的表现情况FAT用相应的取值来描述：

> FAT文件系统通常保存两份FAT表，以防数据丢失。通过正常的系统读 写对FAT1做了更改，那么FAT2也同样被更新。

##### 目录

不管目录文件所占空间为多少簇，一簇为多少字节。系统都会以32个字节为单位进行目录文件所占簇的分配。这32个字节以确定的偏移来定义本目录下的一个文件(或文件夹)的属性，实际上是一个简单的二维表。

### 虚拟磁盘

以文件模拟物理磁盘的结构和分区方式，在vhd文件中模拟磁盘读写，即每次读取或写回一个扇区，而不能直接读取vhd文件中任意位置。
程序运行前打开vhd，相当于磁盘的挂载，退出程序时，关闭vhd。程序运行过程中vhd始终处于打开的状态。
写入和读取文件时，首先检索目录项，找到指定文件后，通过目录中的首簇连接到FAT表，根据FAT链表结构即可找到该文件所处的数据区，建立缓冲区模拟磁头，每次对512字节进行读取或写入，0xFFFF对应文件结束扇区。
文件删除时，并不清除vhd中该文件的全部数据，而是将目录首字节标记为0XE5，表示该文件已删除，相当于回收站功能，可实现文件删除后恢复，当磁盘空间不足时，再将文件内容清除。

### 虚拟文件系统

##### 缓冲文件系统

在vhd模拟的磁盘上建立文件系统，实现文件操作，建立内存缓冲区，程序要操作磁盘中文件的数据，必须要借助缓冲区。
缓冲文件系统会自动在内存中为被操作的文件开辟一块连续的内存单元作为文件缓冲区，当要把数据存储到文件中时，首先把数据写入文件缓冲区，一旦写满了512B，自动把全部数据写入磁盘的一个扇区，然后把缓冲区清空，新的数据继续写入到文件缓冲区。当要从文件读取数据时，自动把文件首扇区的数据导入缓冲区，供程序读取，一旦512B数据都被读取，自动把下一扇区内容导入缓冲区。

##### 文件类型指针

为了将操作与指定文件相关联，定义了如下FILE结构：

```c
typedef struct{
    short   bufsize;             //缓冲区大小
    unsigned char*  buffer;      //缓冲区首地址
    unsigned char*  ptr;         //buf中第一个未读地址
    unsigned int    curp;        //文件指针位置
    unsigned short  firstCluster;   //文件首簇
    unsigned short  curCluster;    //当前缓冲区内的簇号
    unsigned int    filesize;       //文件大小
}   myFILE;
```

FILE的成员指针指向文件缓冲区，通过移动指针实现对文件的操作，除此之外，还记录文件大小等信息。由于结构指针的参数传递效率更高，因此文件指针方式实现：
​	myFILE* 	myfp；

##### 打开文件fopen()

文件打开的实质是把磁盘文件与文件缓冲区对应起来，这样后面的文件读写操作只需使用文件指针即可。执行myfopen，将完成以下步骤的工作：

1. 在磁盘目录中找到指定文件

2. 在内存中分配保存一个myFILE类型结构的单元
3. 在内存中分配文件缓冲区单元512B
4. 返回FILE结构指针

##### 关闭文件fclose()

如果把数据写入文件，首先是写入到文件缓冲区里，只有当写满512B，才会真正写入磁盘文件。如果写的数据不到512B，将不会写入磁盘文件。而通过fclose函数，即使未写满512B，能强制把缓冲区内数据写入磁盘文件，确保写文件操作正确完成。    

##### 文件读写

文件指针中的成员指针标记了当前文件的操作位置，当进行文件读取时，在缓冲区中当前位置写入或读取，同时指针后移，当当前缓冲区操作完成后，将根据FAT表找到文件的下一扇区，并将该扇区512B加载到缓冲区中，继续进行后续操作。
这里实现了以下文件读写操作：

##### 其他相关函数

重定位文件首函数 `void myrewind(myFILE* myfp)`

指针移动控制函数 `void myfseek(myFILE* myfp, int offset)`

获取指针当前位置函数 `int myftell(myFILE* myfp)`

文件末尾检测函数 `bool myfeof(myFILE* myfp)`







