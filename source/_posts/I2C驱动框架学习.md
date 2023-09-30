---
title: I2C驱动框架学习
date: 2023-09-28 21:43:49
tags:
typora-root-url: ./..
---

# I2C通信：
**I2C集成电路总线是一种串行的通信总线，使用主从架构**
特点：
只需要两条双向总线（SDA串行数据线、SCL串行时钟线）
所有组件之间都存在简单的主从关系，连接到总线的每个设备都可以通过唯一地址进行软件寻址。
I2C是真正的多主设备总线，可以提供仲裁和冲突检测。

### CAN总线仲裁：
CAN总线采用的是一种叫做“载波监测，多主掌控／冲突避免”（CSMA／CA）的通信模式。这种总线仲裁方式允许总线上的任何一个设各都有机会取得总线的控制权并向外发送数据。如果在同一时刻有2个或2个以上的设各要求发送数据，就会产生总线冲突，CAN总线能够实时地检测这些冲突并对其进行仲裁，从而使具有高优先级的数据不受任何损坏地传输。
当总线处于空闲状态时呈隐性电平，此时任何节点都可以向总线发送显性电平作为帧的开始。如果2个或2个以上同时发送就会产生竞争。CAN总线解决竞争的方法同以太网的CSMA／CD方法基本相似。此外，CAN总线做了改进并采用CSMA／CA访问总线，按位对标识符进行仲裁。各节点在向总线发送电平的同时，也对总线上的电平读取，并与自身发送的电平进行比较，如果电平相同继续发送下一位，不同则停止发送退出总线竞争。剩余的节点继续上述过程，直到总线上只剩下1个节点发送的电平，总线竞争结束，优先级高的节点获得总线的控制权。
CAN总线以报文为单位进行数据传输，具有最小二进制数的标识符的节点具有最高的优先级。这种优先级一旦在系统设计时确定就不能随意地更改，总线读取产生的冲突主要靠这些位仲裁解决。
![](/imgs/I2C驱动框架学习/CAN总线节点访问过程.gif)
如图所示，节点A和节点B的标识符的第lO、9、8位电平相同，因此两个节点侦听到的信息和它们发出的信息相同。第7位节点B发出一个“1”，但从节点上接收到的消息却是“0”，说明有更高优先级的节点占用总线发送消息。节点B会退出发送处于单纯监听方式而不发送数据；节点A成功发送仲裁位从而获得总线的控制权，继而发送全部消息。总线中的信号持续跟踪最后获得总线控制权发出的报文，本例中节点A的报文将被跟踪。这种非破坏性位仲裁方法的优点在于，在网络最终确定哪个节点被传送前，报文的起始部分已经在网络中传输了，因此具有高优先级的节点的数据传输没有任何延时。在获得总线控制权的节点发送数据过程中，其他节点成为报文的接收节点，并且不会在总线再次空闲之前发送报文。

### 数据传输协议：
主设备和从设备进行数据传输时，数据通过一条SDA数据线在主从设备之间传输0和1的串行数据。串行数据的结构分为：开始条件，地址位，读写位，应答位，数据位，停止条件。

#### 开始条件：
主设备要开始通信时发送开始信号，执行：
将SDA线从高压电平转换到低压电平
将SCL从高电平切换成低压电平

**地址位**：

主机向从机发送/接收数据，需要发送对应的从机地址，然后匹配总线上挂载的从机地址。

**读写位**：

指定数据传输方向：主-->从，该位为0。从-->主，该位为1。

**ACK**/**NACK**：

主机每次发送完数据之后会等待从设备的应答信号ACK。
如果从设备发送应答信号ACK，则SDA会被拉低；
若没有应答信号NACK，则SDA会输出为高电平，这过程会引起主设备发生重启或者停止；

**数据块**：传输数据总共有8位，由发送方设置。将数据位传输到接收方，发送后会紧跟一个ACK/NACK位，如果接收器成功收到数据，则置为0，否则保持逻辑1。重复发送直到数据传送完。

**停止条件**：先将SDA线从低电压电平切换到高电压电平；
再将SCL线从高电平拉到低电平。

### 架构层次分类

![](/imgs/I2C驱动框架学习/I2C驱动框架.png)

##### 第一层：

提供i2c adapter的硬件驱动，探测、初始化i2c adapter（如申请i2c的io地址和中断号），驱动soc控制的i2c adapter在硬件上产生信号（start、stop、ack）以及处理i2c中断。覆盖图中的硬件实现层

##### 第二层：

提供i2c adapter的algorithm，用具体适配器的xxx_xferf()函数来填充i2c_algorithm的master_xfer函数指针，并把赋值后的i2c_algorithm再赋值给i2c_adapter的algo指针。覆盖图中的访问抽象层、i2c核心层

###### 第三层：

实现i2c设备驱动中的i2c_driver接口，用具体的i2c device设备的attach_adapter()、detach_adapter()方法赋值给i2c_driver的成员函数指针。实现设备device与总线（或者叫adapter）的挂接。覆盖图中的driver驱动层

##### 第四层：

实现i2c设备所对应的具体device的驱动，i2c_driver只是实现设备与总线的挂接，而挂接在总线上的设备则是千差万别的，eeprom和ov2715显然不是同一类的device，所以要实现具体设备device的write()、read()、ioctl()等方法，赋值给file_operations，然后注册字符设备（多数是字符设备）。覆盖图中的driver驱动层

第一层和第二层又叫i2c总线驱动(bus)，第三第四属于i2c设备驱动(device driver)。在linux驱动架构中，几乎不需要驱动开发人员再添加bus，因为linux内核几乎集成所有总线bus，如usb、pci、i2c等等。并且总线bus中的【与特定硬件相关的代码】已由芯片提供商编写完成，例如TI davinci平台i2c总线bus与硬件相关的代码在内核目录/drivers/i2c/buses下的i2c-davinci.c源文件中；而三星的s3c-2440平台i2c总线bus为/drivers/i2c/buses/i2c-s3c2410.c

第三第四层又叫设备驱动层与特定device相干的就需要驱动工程师来实现了。

#### 具体分析
i2c_adapter与i2c_client的关系与i2c硬件体系中设配器与设备的关系一致，即i2c_client依附于i2c_adapter，由于一个适配器上可以连接多个i2c设备device，所以相应的，i2c_adapter也可以被多个i2c_client依附，在i2c_adapter中包含i2c_client的链表。同一类的i2c设备device对应一个驱动driver。driver与device的关系是一对多的关系。

看一下这几个重要的结构体，分别是i2c_driver i2c_client i2c_adapter，也可以先忽略他们，待会回过头来看会更容易理解

##### i2c_driver

```c
struct i2c_driver {
	int id;
	unsigned int class;
 
	int (*attach_adapter)(struct i2c_adapter *);
	int (*detach_adapter)(struct i2c_adapter *);
 
	int (*detach_client)(struct i2c_client *);
 
	int (*command)(struct i2c_client *client,unsigned int cmd, void *arg);
	struct device_driver driver;
	struct list_head list;
};
```
};
##### i2c_client

```c
struct i2c_client {
	unsigned int flags;		/* div., see below		*/
	unsigned short addr;		/* chip address - NOTE: 7bit 	*/
					/* addresses are stored in the	*/
					/* _LOWER_ 7 bits		*/
	struct i2c_adapter *adapter;	/* the adapter we sit on	*/
	struct i2c_driver *driver;	/* and our access routines	*/
	int usage_count;		/* How many accesses currently  */
					/* to the client		*/
	struct device dev;		/* the device structure		*/
	struct list_head list;
	char name[I2C_NAME_SIZE];
	struct completion released;
};
```

##### i2c_adapter

```c
struct i2c_adapter {
	struct module *owner;
	unsigned int id;
	unsigned int class;
	struct i2c_algorithm *algo;/* the algorithm to access the bus	*/
	void *algo_data;
 
	/* --- administration stuff. */
	int (*client_register)(struct i2c_client *);
	int (*client_unregister)(struct i2c_client *);
 
	/* data fields that are valid for all devices	*/
	struct mutex bus_lock;
	struct mutex clist_lock;
 
	int timeout;
	int retries;
	struct device dev;		/* the adapter device */
	struct class_device class_dev;	/* the class device */
 
	int nr;
	struct list_head clients;
	struct list_head list;
	char name[I2C_NAME_SIZE];
	struct completion dev_released;
	struct completion class_dev_released;
};
```
##### i2c_algorithm

```c
struct i2c_algorithm {
	int (*master_xfer)(struct i2c_adapter *adap,struct i2c_msg *msgs, 
	                   int num);
	int (*slave_send)(struct i2c_adapter *,char*,int);
	int (*slave_recv)(struct i2c_adapter *,char*,int);
	u32 (*functionality) (struct i2c_adapter *);
};
```

##### 【i2c_adapter与i2c_algorithm】
i2c_adapter对应与物理上的一个适配器，而i2c_algorithm对应一套通信方法，一个i2c适配器需要i2c_algorithm中提供的（i2c_algorithm中的又是更下层与硬件相关的代码提供）通信函数来控制适配器上产生特定的访问周期。缺少i2c_algorithm的i2c_adapter什么也做不了，因此i2c_adapter中包含其使用i2c_algorithm的指针。

i2c_algorithm中的关键函数master_xfer()用于产生i2c访问周期需要的start stop ack信号，以i2c_msg（即i2c消息）为单位发送和接收通信数据。i2c_msg也非常关键，调用驱动中的发送接收函数需要填充该结构体

 	/*
 	 * I2C Message - used for pure i2c transaction, also from /dev interface
 	 */
 	struct i2c_msg {
 		__u16 addr;	/* slave address			*/
 	 	__u16 flags;		
 	 	__u16 len;		/* msg length				*/
 	 	__u8 *buf;		/* pointer to msg data			*/
 	};

##### 【i2c_driver和i2c_client】
i2c_driver对应一套驱动方法，其主要函数是attach_adapter()和detach_client()，i2c_client对应真实的i2c物理设备device，每个i2c设备都需要一个i2c_client来描述，i2c_driver与i2c_client的关系是一对多。一个i2c_driver上可以支持多个同等类型的i2c_client.

##### 【i2c_adapter和i2c_client】

i2c_adapter和i2c_client的关系与i2c硬件体系中适配器和设备的关系一致，即i2c_client依附于i2c_adapter,由于一个适配器上可以连接多个i2c设备，所以i2c_adapter中包含依附于它的i2c_client的链表。
从图1图2中都可以看出，linux内核对i2c架构抽象了一个叫核心层core的中间件，它分离了设备驱动device driver和硬件控制的实现细节（如操作i2c的寄存器），core层不但为上面的设备驱动提供封装后的内核注册函数，而且还为小面的硬件时间提供注册接口（也就是i2c总线注册接口），可以说core层起到了承上启下的作用。

我们先看一下i2c-core为外部提供的核心函数（选取部分），i2c-core对应的源文件为i2c-core.c，位于内核目录/driver/i2c/i2c-core.c

```c
EXPORT_SYMBOL(i2c_add_adapter);
EXPORT_SYMBOL(i2c_del_adapter);
EXPORT_SYMBOL(i2c_del_driver);
EXPORT_SYMBOL(i2c_attach_client);
EXPORT_SYMBOL(i2c_detach_client);

EXPORT_SYMBOL(i2c_transfer);
```

i2c_transfer()函数，i2c_transfer()函数本身并不具备驱动适配器物理硬件完成消息交互的能力，它只是寻找到i2c_adapter对应的i2c_algorithm，并使用i2c_algorithm的master_xfer()函数真正的驱动硬件流程，代码清单如下，不重要的已删除。

```c
int i2c_transfer(struct i2c_adapter * adap, struct i2c_msg *msgs, int num)
{
	int ret;
	if (adap->algo->master_xfer) {//如果master_xfer函数存在，则调用，否则返回错误
		ret = adap->algo->master_xfer(adap,msgs,num);//这个函数在硬件相关的代码中给algorithm赋值
		return ret;
	} else {
		return -ENOSYS;
	}
}
```

当一个具体的client被侦测到并被关联的时候，设备和sysfs文件将被注册。相反的，在client被取消关联的时候，sysfs文件和设备也被注销，驱动开发人员需开发i2c设备驱动时，需要调用下列函数。程序清单如下

```c
int i2c_attach_client(struct i2c_client *client)
{
	...
	device_register(&client->dev);
	device_create_file(&client->dev, &dev_attr_client_name);
	...
	return 0;
}
```

```c
int i2c_detach_client(struct i2c_client *client)
{
	...
	device_remove_file(&client->dev, &dev_attr_client_name);
	device_unregister(&client->dev);
	...
	return res;
}
```

i2c_add_adapter()函数和i2c_del_adapter()在i2c-davinci.c中有调用

```c
/* -----
 * i2c_add_adapter is called from within the algorithm layer,
 * when a new hw adapter registers. A new device is register to be
 * available for clients.
 */
int i2c_add_adapter(struct i2c_adapter *adap)
{
	...
	device_register(&adap->dev);
	device_create_file(&adap->dev, &dev_attr_name);
	...
	/* inform drivers of new adapters */
	list_for_each(item,&drivers) {
		driver = list_entry(item, struct i2c_driver, list);
		if (driver->attach_adapter)
			/* We ignore the return code; if it fails, too bad */
			driver->attach_adapter(adap);
	}
	...
}
```

```c
int i2c_del_adapter(struct i2c_adapter *adap)
{
	...
	list_for_each(item,&drivers) {
		driver = list_entry(item, struct i2c_driver, list);
		if (driver->detach_adapter)
			if ((res = driver->detach_adapter(adap))) {
			}
	}
	...
	list_for_each_safe(item, _n, &adap->clients) {
		client = list_entry(item, struct i2c_client, list);
 
		if ((res=client->driver->detach_client(client))) {
 
		}
	}
	...
	device_remove_file(&adap->dev, &dev_attr_name);
	device_unregister(&adap->dev);
 
}
```

i2c-davinci.c是实现与硬件相关功能的代码集合，这部分是与平台相关的，也叫做i2c总线驱动，这部分代码是这样添加到系统中的

```c
static struct platform_driver davinci_i2c_driver = {
	.probe		= davinci_i2c_probe,
	.remove		= davinci_i2c_remove,
	.driver		= {
		.name	= "i2c_davinci",
		.owner	= THIS_MODULE,
	},
};
 
/* I2C may be needed to bring up other drivers */
static int __init davinci_i2c_init_driver(void)
{
	return platform_driver_register(&davinci_i2c_driver);
}
subsys_initcall(davinci_i2c_init_driver);
 
static void __exit davinci_i2c_exit_driver(void)
{
	platform_driver_unregister(&davinci_i2c_driver);
}
module_exit(davinci_i2c_exit_driver);
```

并且，i2c适配器控制硬件发送接收数据的函数在这里赋值给i2c-algorithm，i2c_davinci_xfer稍加修改就可以在裸机中控制i2c适配器

```c
static struct i2c_algorithm i2c_davinci_algo = {
	.master_xfer	= i2c_davinci_xfer,
	.functionality	= i2c_davinci_func,
};
```

然后在davinci_i2c_probe函数中，将i2c_davinci_algo添加到添加到algorithm系统中

```c
adap->algo = &i2c_davinci_algo;
```


### 梳理图
有时候代码比任何文字描述都来得直接，但是过多的代码展示反而让人觉得枯燥。这个时候，需要一幅图来梳理一下上面的内容:


![](/imgs/I2C驱动框架学习/I2C适配器Dm368硬件.png)

linux内核和芯片提供商为我们的的驱动程序提供了 i2c驱动的框架，以及框架底层与硬件相关的代码的实现。剩下的就是针对挂载在i2c两线上的i2c设备了device，如at24c02，例如ov2715，而编写的具体设备驱动了，这里的设备就是硬件接口外挂载的设备，而非硬件接口本身（soc硬件接口本身的驱动可以理解为总线驱动）。

在理解了i2c驱动架构后，我们接下来再作两方面的分析工作：一是具体的i2c设备ov2715驱动源码分析，二是davinci平台的i2c总线驱动源码。

#### ov2715设备i2c驱动源码分析
ov2715为200万的CMOS Sensor，芯片的寄存器控制通过i2c接口完成，i2c设备地址为0x6c，寄存器地址为16位两个字节，寄存器值为8位一个字节，可以理解为一般的字符设备。
该驱动程序并非只能用于ov2715，因此源码中存在支持多个设备地址的机制。
该字符设备的用到的结构体有两个，如下

```c
typedef struct {

  int devAddr;

  struct i2c_client client;   //!< Data structure containing general access routines.
  struct i2c_driver driver;   //!< Data structure containing information specific to each client.

  char name[20];
  int nameSize;
  int users;

} I2C_Obj;
```

```c
#define I2C_DEV_MAX_ADDR  (0xFF)
#define I2C_TRANSFER_BUF_SIZE_MAX   (256)
typedef struct {
 
  struct cdev cdev;             /* Char device structure    */
  int     major;
  struct semaphore semLock;
    
  I2C_Obj *pObj[I2C_DEV_MAX_ADDR];
 
  uint8_t reg[I2C_TRANSFER_BUF_SIZE_MAX];
  uint16_t reg16[I2C_TRANSFER_BUF_SIZE_MAX];
  uint8_t buffer[I2C_TRANSFER_BUF_SIZE_MAX*4];
  
} I2C_Dev;
```

一个I2C_Obj描述一个设备，devAddr保存该设备的地址，I2C_Obj内嵌到结构体I2C_Dev，I2C_Dev管理该驱动所支持的所有设备，尽管支持多个设备，但i2c适配器只有一个，因此需要一个信号量semLock来保护该共享资源，同时只能向一个设备读写数据。成员变量cdev是我们所熟知的，每个字符设备驱动中几乎总会有一个结构体包含它，major用于保存该驱动的主设备编号，reg数组为寄存器地址为8位的寄存器地址缓冲区，reg16为寄存器地址为16的寄存器地址缓冲区。同时可以读写多个寄存器地址的值。buffer为读写的寄存器值
使用I2C_Dev构建一个全局变量gI2C_dev，在驱动的多个地方均需要它。
下面先从字符设备的基本框架入手，然后深入该驱动的细节部分。
首先是该字符设备的初始化和退出函数

```c
int I2C_devInit(void)
{
  int     result, i;
  dev_t   dev = 0;
 
  result = alloc_chrdev_region(&dev, 0, 1, I2C_DRV_NAME);//分配字符设备空间
  
  for(i=0; i<I2C_DEV_MAX_ADDR; i++)
  {
    gI2C_dev.pObj[i]=NULL;
  }
 
  gI2C_dev.major = MAJOR(dev);//保存设备主编号
  sema_init(&gI2C_dev.semLock, 1);//信号量初始化
  cdev_init(&gI2C_dev.cdev, &gI2C_devFileOps);//使用gI2C_devFileOps初始化该字符设备，gI2C_devFileOps见下文
  gI2C_dev.cdev.owner = THIS_MODULE;//常规赋值
 gI2C_dev.cdev.ops = &gI2C_devFileOps;//常规赋值 result = cdev_add(&gI2C_dev.cdev, dev, 1);//添加设备到字符设备中 return result;}void I2C_devExit(void){ dev_t devno = MKDEV(gI2C_dev.major, 0); cdev_del(&gI2C_dev.cdev);//从字符设备中删除该设备 unregister_chrdev_region(devno, 1);//回收空间}
gI2c_devFileOps全局变量，驱动初始化会用到该结构体变量
struct file_operations gI2C_devFileOps = {
  .owner = THIS_MODULE,
  .open = I2C_devOpen,
  .release = I2C_devRelease,
  .ioctl = I2C_devIoctl,
};
```

该驱动只实现了三个函数,open,release和ioctl，对于i2c设备来说，这已经足够了。
在I2C_devOpen和I2C_devOpen中并没有做实际的工作，重要的工作均在I2C_devIoctl这个ioctl中完成。I2C_devIoctl代码展示（将影响结构条理的代码去掉，稍后在做详细分析）

```c
int I2C_devIoctl(struct inode *inode, struct file *filp, unsigned int cmd, unsigned long arg)
{
  I2C_Obj *pObj;
  int status=0;
  I2C_TransferPrm transferPrm;
  
  pObj = (I2C_Obj *)filp->private_data;
 
  if(!I2C_IOCTL_CMD_IS_VALID(cmd))
    return -1;
  cmd = I2C_IOCTL_CMD_GET(cmd);//cmd命令转换，防止混淆，具体原因参见上一篇文章：ioctl中的cmd
 
  down_interruptible(&gI2C_dev.semLock);      //信号量down
  
  switch(cmd)
  {
    case I2C_CMD_SET_DEV_ADDR://命令1，设置设备地址
      filp->private_data = I2C_create(arg);
 
    case I2C_CMD_WRITE:  //命令2，写寄存器值
      
      status = copy_from_user(&transferPrm, (void *)arg, sizeof(transferPrm));
      ...
            
      break;
    case I2C_CMD_READ:  //命令3，读寄存器值
    
      status = copy_from_user(&transferPrm, (void *)arg, sizeof(transferPrm));
      ...
      
      break;
    default:
      status = -1;
      break;    
  }
 
  up(&gI2C_dev.semLock);      //信号量up
 
  return status;
}
```

以上三个命令中最重要最复杂的是第一个I2C_CMD_SET_DEV_ADDR，设置设备地址，之所以重要和复杂，因为在I2C_create()函数中，将通过i2c-core提供的函数把该驱动程序和底层的i2c_adapter联系起来。下面是I2C_create()函数源码

```c
void *I2C_create(int devAddr) {
 
    int ret;
    struct i2c_driver *driver;
    struct i2c_client *client = client;
    I2C_Obj *pObj;
 
    devAddr >>= 1;
    
    if(devAddr>I2C_DEV_MAX_ADDR)  //变量合法性判断
      return NULL;
   
    if(gI2C_dev.pObj[devAddr]!=NULL) {	//变量合法性判断，如果该地址的设备已经创建，则调过，防止上层错误调用
      // already allocated, increment user count, and return the allocated handle
      gI2C_dev.pObj[devAddr]->users++;
      return gI2C_dev.pObj[devAddr];
    }
    
    pObj = (void*)kmalloc( sizeof(I2C_Obj), GFP_KERNEL); //为pObj分配空间
    gI2C_dev.pObj[devAddr] = pObj;  //将分配的空间地址保存在全局变量里
    memset(pObj, 0, sizeof(I2C_Obj));
  
    pObj->client.adapter = NULL;
    pObj->users++;    //用户基数，初始化为0，当前设为1
    pObj->devAddr = devAddr;  //保存设备地址
    
    gI2C_curAddr = pObj->devAddr;  //gI2C_curAddr为全局的整型变量，用于保存当前的设备地址
    driver = &pObj->driver;  //将成员变量driver单独抽取出来，因为线面要使用driver来初始化驱动
 
    pObj->nameSize=0;//i2c设备名称，注意，这里不是在/dev下面的设备节点名
    pObj->name[pObj->nameSize++] = 'I';
    pObj->name[pObj->nameSize++] = '2';
    pObj->name[pObj->nameSize++] = 'C';
    pObj->name[pObj->nameSize++] = '_';   
    pObj->name[pObj->nameSize++] = 'A' + ((pObj->devAddr >> 0) & 0xF);
    pObj->name[pObj->nameSize++] = 'B' + ((pObj->devAddr >> 4) & 0xF);
    pObj->name[pObj->nameSize++] = 0;
 
    driver->driver.name = pObj->name; //保存刚才设置的name
    driver->id = I2C_DRIVERID_MISC;
    driver->attach_adapter = I2C_attachAdapter;   //这个很重要，将驱动连接到i2c适配器上，在后面分析
    driver->detach_client = I2C_detachClient;	//这个很重，在后面分析
 
    if((ret = i2c_add_driver(driver)))	//使用i2c-core（i2c_register_driver函数）的接口，注册该驱动，i2c_add_driver实质调用了driver_register()
    {
        printk( KERN_ERR "I2C: ERROR: Driver registration failed (address=%x), module not inserted.\n", pObj->devAddr);
    }
 
    if(ret<0) {
 
      gI2C_dev.pObj[pObj->devAddr] = NULL;
      kfree(pObj);    
      return NULL;
    }
    return pObj;
}
```

其他两个命令是I2C_CMD_WRITE和I2C_CMD_READ，这个比较简单，只需设置寄存器地址的大小以及寄存器值的大小，然后通过i2c-core 提供的i2c_transfer()函数发送即可。例如I2C_wirte()

```c
int I2C_write(I2C_Obj *pObj, uint8_t *reg, uint8_t *buffer, uint8_t count, uint8_t dataSize)
{
  uint8_t i;
  int err;
  struct i2c_client *client;
	struct i2c_msg msg[1];
	unsigned char data[8];
 
  if(pObj==NULL)
    return -ENODEV;
 
  client = &pObj->client;//得到client信息
  if(!client->adapter)
    return -ENODEV;  
  
  if(dataSize<=0||dataSize>4)
    return -1;
    
  for(i=0; i<count; i++) {
  
    msg->addr = client->addr;//设置要写的i2c设备地址
    msg->flags= 0;//一直为0
    msg->buf  = data;//date为准备i2c通信的缓冲区，这个缓冲区除了不包含设备地址外，要包括要目标寄存器地址，和要写入该寄存器的值
		
    data[0] = reg[i];//寄存器地址赋值
		
    if(dataSize==1) {//寄存器值长度为1
       data[1]  = buffer[i];//寄存器值赋值
       msg->len = 2;  	//设置data长度为2	
    }	else if(dataSize==2) {//寄存器值长度为2
       data[1] = buffer[2*i+1];
       data[2] = buffer[2*i];
       msg->len = 3;
    } 
    err = i2c_transfer(client->adapter, msg, 1);//调用i2c-core中的i2c_transfer发送i2c数据
    if( err < 0 )
      return err;
    }
  
  return 0;
}
```

重点分析上一段代码void *I2C_create(int devAddr)函数中的i2c_driver结构体部分的代码，下面的代码是从上面I2C_create抽取出来的

```c
    driver->driver.name = pObj->name;
    driver->id = I2C_DRIVERID_MISC;
    driver->attach_adapter = I2C_attachAdapter;
    driver->detach_client = I2C_detachClient;
```
在i2c_driver结构体中针对attach_adapter有这样的说明：

```c
	/* Notifies the driver that a new bus has appeared. This routine
	 * can be used by the driver to test if the bus meets its conditions
	 * & seek for the presence of the chip(s) it supports. If found, it 
	 * registers the client(s) that are on the bus to the i2c admin. via
	 * i2c_attach_client.
	 */
```

意思是通知驱动，i2c适配器已经就绪了，这时可以讲device的driver连接到总线bus上。所以I2C_attachAdapter的作用就是检测client，然后将client连接上来。attach_adapter和detach_client由内核驱动自动调用，我们只需在调用的时候实现必要的功能即可，如下代码展示

```c
int I2C_attachAdapter(struct i2c_adapter *adapter)
{
    return I2C_detectClient(adapter, gI2C_curAddr);
}
 
int I2C_detectClient(struct i2c_adapter *adapter, int address)
{
    I2C_Obj *pObj;
    struct i2c_client *client;
    int err = 0;
    
    if(address > I2C_DEV_MAX_ADDR) {
      printk( KERN_ERR "I2C: ERROR: Invalid device address %x\n", address);        
      return -1;
    }
      
    pObj = gI2C_dev.pObj[address];
    if(pObj==NULL) {
      printk( KERN_ERR "I2C: ERROR: Object not found for address %x\n", address);    
      return -1;
    }
 
    client = &pObj->client;
 
    if(client->adapter)
      return -EBUSY;  /* our client is already attached */
 
    memset(client, 0x00, sizeof(struct i2c_client));
    client->addr = pObj->devAddr;
    client->adapter = adapter;
    client->driver = &pObj->driver;
 
    if((err = i2c_attach_client(client)))
    {
        printk( KERN_ERR "I2C: ERROR: Couldn't attach %s (address=%x)\n", pObj->name, pObj->devAddr);
        client->adapter = NULL;
        return err;
    }
    return 0;
}
```
最终I2C_detectClient()函数调用了i2c-core中的i2c_attach_client，从名字上就能看出什么意思，连接client设备。
当内核驱动准备删除该驱动时会自动调用i2c_driver的成员函数：detech_client，因此我们需要实现删除client设备的函数然后赋值给改函数指针，detech_client的说明如下：

```c
	/* tells the driver that a client is about to be deleted & gives it 
	 * the chance to remove its private data. Also, if the client struct
	 * has been dynamically allocated by the driver in the function above,
	 * it must be freed here.
	 */
```

下面是detech_client调用的函数代码清单，该函数最终调用了i2c-core提供的i2c_detach_client，用于取消client设备的连接

```c
int I2C_detachClient(struct i2c_client *client)
{
    int err;
if(!client->adapter)
    return -ENODEV; /* our client isn't attached */
 
if((err = i2c_detach_client(client))) {
    printk( KERN_ERR "Client deregistration failed (address=%x), client not detached.\n", client->addr);
    return err;
}
 
client->adapter = NULL;
 
return 0;
```
}








<small>misslyh20080512202305122023080719980106202309281520825879280398965</small>
