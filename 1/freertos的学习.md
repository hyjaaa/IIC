#                                  freertos



## RTOS的概念

### 1.裸机开发：

​	在传统的开发当中，我们一般使用裸机进行开发，但是随着学习的深入我发现裸机开发暴露出来的问题很多：
**1、程序并发工作效率低**：

​	在51和32上面写程序的时候，无法避免在一个大的while（1）里面进行循环，一堆delay函数导致程序运行起来很慢，浪费CPU资源。

**2、模块化导致程序很复杂开发进行困难**：	由于程序都执行在一个大的while函数里面，往往出了bug很难找出来问题出现在哪里。一个任务出现了问题，另一个任务也不能执行，开发的时候感到十分困难。

**3、功能多的时候实时性很差**

 	程序在一个大的while循环里面，从上到下一条条执行，任务不能同时发生。就好比如：吃饭的时候，不能刷手机，只能吃饭，等到吃饭完后才能刷手机。

### 2.RTOS的优势：

为了解决这些问题我们这里引入了RTOS，RTOS的意思是：Real-time operating system，实时操作系统。

RTOS和裸机的主要区别在于:

1. **任务调度:**

- RTOS提供了任务(线程)的概念和任务调度功能。可以实现多任务并行执行。

- 裸机只有一个任务,需要开发人员自行实现任务切换或者协作多任务。

2. **时间管理:**

- RTOS内部采用周期中断实现时间管理,支持时间片、延时等函数。 

- 裸机需要开发人员通过定时器中断等方式自己管理时间。

3. **同步机制:**

- RTOS提供信号量、事件标志、消息队列等同步原语来控制任务间的同步和通信。

- 裸机需要通过共享内存、管道等自行实现同步机制。

4. **内存管理:**

- RTOS提供内存池管理内存,自动分配释放内存。

- 裸机需要手动管理内存,如使用链表实现内存池。

### 3.简单程序举例：

​	

```c
// 经典单片机程序
void main()
{
	while (1)
    {
        喂一口饭();
        回一个信息();
    }
}
------------------------------------------------------
// RTOS程序    
喂饭()
{
    while (1)
    {
        喂一口饭();
    }
}

回信息()
{
    while (1)
    {
        回一个信息();
    }
}

void main()
{
    create_task(喂饭);
    create_task(回信息);
    start_scheduler();
    while (1)
    {
        sleep();
    }
}
```



## freertos之任务的初步认识

### 1.什么是任务

在freertos 中，任务就是一个运行着的函数，原型如下：

```c
void ATaskFunction( void *pvParameters );
```

要注意的是：
	这个函数不能返回
	同一个函数，可以用来创建多个任务；换句话说，多个任务可以运行同一
个函数
	函数内部，尽量使用局部变量：

- ​	**每个任务都有自己的栈**

- ​	每个任务运行这个函数时：

  ​		  任务 A 的局部变量放在任务 A 的栈里、任务 B 的局部变量放在任务 B的栈里

​			      不同任务的局部变量，有自己的副本

- 函数使用全局变量、静态变量的话：

  ​	只有一个副本：多个任务使用的是同一个副本

  ​	要防止冲突(后续会讲)  

### 2.任务的创建

（这里我们只讨论动态任务的创建，静态任务因为不常用所以不在这里展示）

#### 1.函数原型：

![image-20231202224211543](C:\Users\20215\AppData\Roaming\Typora\typora-user-images\image-20231202224211543.png)

#### 2.参数解释：

| 参数          | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| pvTaskCode    | 函数指针，可以简单地认为任务就是一个 C 函数。它稍微特殊一点：永远不退出，或者退出时要调用"vTaskDelete(NULL)" |
| pcName        | 任务的名字， FreeRTOS 内部不使用它，仅仅起调试作用。<br />长度为： configMAX_TASK_NAME_LEN |
| usStackDepth  | 每个任务都有自己的栈，这里指定栈大小。单位是 word，比如传入 100，表示栈大小为 100 word，也就是 400 字节。最大值为 uint16_t 的最大值。怎么确定栈的大小，并不容易，很多时候是估计。精确的办法是看反汇编码。 |
| pvParameters  | 调用 pvTaskCode 函数指针时用到： pvTaskCode(pvParameters)    |
| uxPriority    | 优先级范围： 0~(configMAX_PRIORITIES – 1)数值越小优先级越低，如果传入过大的值， xTaskCreate 会把它调整为(configMAX_PRIORITIES – 1) |
| pxCreatedTask | 用来保存 xTaskCreate 的输出结果： task handle。以后如果想操作这个任务，比如修改它的优先级，就需要这个 handle。如果不想使用该 handle，可以传入 NULL。 |
| 返回值        | 成功： pdPASS；<br />失败： errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY(失败原因只有内存不足)注意：文档里都说失败时返回值是 pdFAIL，这不对。pdFAIL 是 0， errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY 是-1。 |

#### 3.示例代码：

```c
void Task1Function(void * param)
{
	while (1)
	{
		printf("1");
	}
}

void Task2Function(void * param)
{
	while (1)
	{
		printf("2");
	}
}


/*-----------------------------------------------------------*/

int main( void )
{
	TaskHandle_t xHandleTask1;
		
#ifdef DEBUG
  debug();
#endif

	prvSetupHardware();

	printf("Hello, world!\r\n");

	xTaskCreate(Task1Function, "Task1", 100, NULL, 1, &xHandleTask1);
	xTaskCreate(Task2Function, "Task2", 100, NULL, 1, NULL);

	/* Start the scheduler. */
	vTaskStartScheduler();

	/* Will only get here if there was not enough heap space to create the
	idle task. */
	return 0;
}
```

打开仿真的串口显示，结果如图显示：

![image-20231202225754939](C:\Users\20215\AppData\Roaming\Typora\typora-user-images\image-20231202225754939.png)

任务1和任务2交替执行

### 3.任务的删除

```c
void vTaskDelete( TaskHandle_t xTaskToDelete );
```

参数说明：

​					任 务 句 柄 ， 使 用 xTaskCreate 创 建 任 务 时 可 以 得 到 一 个 句 柄 。也可传入 NULL，这表示删除自己。

​					

怎么删除任务？举个不好的例子：



- 自杀： vTaskDelete(NULL)
- 被杀：别的任务执行 vTaskDelete(pvTaskCode)， pvTaskCode 是自己的句柄 
- 杀人：执行 vTaskDelete(pvTaskCode)， pvTaskCode 是别的任务的句柄

**强调一下任务不能重复删除，否则程序会卡死**

在默认的调度机制下，高优先级的任务先执行，如果高优先级的任务不主动放弃，低优先级的任务永远没办法执行

任务优先级相同则任务交替执行

### 4.任务状态

#### 1.阻塞状态：

在日常生活的例子中，母亲在电脑前跟同事沟通时，如果同事一直没回复，那么母亲的工作就被卡住了、被堵住了、处于阻塞状态(Blocked)。重点在于：母亲在**等待**。  

很明显任务在执行的时候使用了vtaskdelay（）会马上进入阻塞态



#### 2.暂停状态（或者挂起态）：

FreeRTOS 中的任务也可以进入暂停状态，唯一的方法是通过 vTaskSuspend 函数。函数原型如下

```c
void vTaskSuspend( TaskHandle_t xTaskToSuspend );
```

  参数 xTaskToSuspend 表示要暂停的任务，如果为 NULL，表示暂停自己。



要退出暂停状态，只能由别人来操作： 

-  	别的任务调用： vTaskResume 
-  	中断程序调用： xTaskResumeFromISR（后面我们再介绍中断的使用，这里暂时不进行说明）

#### 3.就绪态：

这个任务完全准备好了，随时可以运行：只是还轮不到它。这时，它就处于就绪态(Ready)  。

#### 4.任务转化图：

![image-20231204180617994](C:\Users\20215\AppData\Roaming\Typora\typora-user-images\image-20231204180617994.png)



## freertos之delay函数

### 1.概念：

- vTaskDelay():它使调用任务阻塞指定的tick数,等待task delay period这个时间间隔才能转入就绪状态。

- vTaskDelayUntil():它指定一个绝对时间点(以时间戳表示),调用任务将阻塞直到这个时间戳到达才进入就绪状态。

两者的区别是:

1. vTaskDelay()以时间片tick为单位指定延迟值,具体延迟时间取决于tick的时间长度。

2. vTaskDelayUntil()直接指定一个绝对时间点,不依赖时间片长短,实现了更精确的延迟控制。

使用场景:

- vTaskDelay():用于需要大致延迟的场景,比如定时任务轮转。

- vTaskDelayUntil():需要非常精确延迟的场景,如硬件定时器同步等。

### 2.举例：

可能从概念上来说这样不容易理解，下面我们通过举例来详细说明：

```c
TaskHandle_t xHandleTask1;
TaskHandle_t xHandleTask3;

static int task1flagrun = 0;
static int task2flagrun = 0;
static int task3flagrun = 0;

static int rands[] = {3, 56, 23, 5, 99};

void Task1Function(void * param)
{
	TickType_t tStart = xTaskGetTickCount();
	int i = 0;
	int j = 0;
	
	while (1)
	{
		
		task1flagrun = 1;
		task2flagrun = 0;
		task3flagrun = 0;
        //下面注释的内容可以忽略
/*
		for (i = 0; i < rands[j]; i++)
			printf("1");

		j++;
		if (j == 5)
			j = 0;
*/
		vTaskDelay(20);

//		vTaskDelayUntil(&tStart, 20);

	}
}

void Task2Function(void * param)
{
	while (1)
	{
		task1flagrun = 0;
		task2flagrun = 1;
		task3flagrun = 0;
		printf("2");
	}
}

void Task3Function(void * param)
{
	while (1)
	{
		task1flagrun = 0;
		task2flagrun = 0; 
		task3flagrun = 1;
		printf("3");
	}
}


/*-----------------------------------------------------------*/

StackType_t xTask3Stack[100];
StaticTask_t xTask3TCB;

StackType_t xIdleTaskStack[100];
StaticTask_t xIdleTaskTCB;


/*
 * The buffers used here have been successfully allocated before (global variables)
 */
void vApplicationGetIdleTaskMemory( StaticTask_t ** ppxIdleTaskTCBBuffer,
                                    StackType_t ** ppxIdleTaskStackBuffer,
                                    uint32_t * pulIdleTaskStackSize )
{
    *ppxIdleTaskTCBBuffer = &xIdleTaskTCB;
    *ppxIdleTaskStackBuffer = xIdleTaskStack;
    *pulIdleTaskStackSize = 100;
}


int main( void )
{
		
#ifdef DEBUG
  debug();
#endif

	prvSetupHardware();

	printf("Hello, world!\r\n");

	xTaskCreate(Task1Function, "Task1", 100, NULL, 2, &xHandleTask1);
	xTaskCreate(Task2Function, "Task2", 100, NULL, 1, NULL);

	xHandleTask3 = xTaskCreateStatic(Task3Function, "Task3", 100, NULL, 1, xTask3Stack, &xTask3TCB);

	/* Start the scheduler. */
	vTaskStartScheduler();

	/* Will only get here if there was not enough heap space to create the
	idle task. */
	return 0;
}
```



我们将task1flagrun，  task2flagrun ， task3flagrun 这三个变量添加到逻辑分析仪里面进行观察


​					

​																					使用vTaskDelayUntil(&tStart, 20)的：

![image-20231204192703785](C:\Users\20215\AppData\Roaming\Typora\typora-user-images\image-20231204192703785.png)

​																						使用vTaskDelay(20)的：

![image-20231204192723923](C:\Users\20215\AppData\Roaming\Typora\typora-user-images\image-20231204192723923.png)

所以我们得出以下的结果：

- vTaskDelay()：从任务结束到下一个任务开始间隔的时间是几乎一样的
- vTaskDelayUntil()：从任务开始到下一个任务开始间隔的时间是几乎一样的

## freertos之空闲任务及其钩子函数

### 1.空闲任务：

一个良好的程序，它的任务都是事件驱动的：平时大部分时间处于阻塞状态。有可能我们自己创建的所有任务都无法执行，但是调度器必须能找到一个可以运行的任务：所以，我们要提供空闲任务。在使用vTaskStartScheduler()

函数来创建、启动调度器时，这个函数内部会创建空闲任务：

首先：

**空闲任务优先级为 0：它不能阻碍用户任务运行**

**空闲任务要么处于就绪态， 要么处于运行态，永远不会阻塞**

FreeRTOS中的空闲任务主要有以下几个作用:

- 空闲处理:主要负责系统中暂时没有其他任务运行时的处理,如休眠模式进入等。

- 内存管理:空闲任务定期扫描任务控制块,回收已删除任务的内存资源。

  这里就不得不提到vTaskDelete这个函数了

  - 自杀的任务：在空闲任务中完成清理工作，比如释放内存(都自杀了，怎么清理自己的尸体? 由别人来做)

  - 非自杀的任务：在vTaskDelete内部完成清理工作(凶手执行清理工作)

    以下是通过查阅的相关资料的拓展：如果一个任务A通过调用vTaskDelete删除另一个任务B,那么任务B的内存栈等资源会由谁来进行清理呢?

    对于这个问题,FreeRTOS的规定是:

    从V9.0版本开始，如果一个任务删除另外一个任务，**被删除任务的堆栈和TCB立即释放**。如果一个任务删除自己，则任务的堆栈和TCB和以前一样，通过空闲任务删除。所以空闲任务开始就会检查是否有任务删除了自己，如果有的话，空闲任务负责删除这个任务的TCB和堆栈空间。

    

- 系统钩子:空闲任务中包含了一些系统钩子函数,如idle hook、tick hook等。作用：
  - 执行一些低优先级的、后台的、需要连续执行的函数
  - 测量系统的空闲时间：空闲任务能被执行就意味着所有的高优先级任务都停止了，所以测量空闲任
    务占据的时间，就可以算出处理器占用率。  
  - 让系统进入省电模式：空闲任务能被执行就意味着没有重要的事情要做，当然可以进入省电模式
    了。  

  - 绝对不能导致任务进入Blocked、Suspended状态
  - 如果你会使用 vTaskDelete() 来删除任务，那么钩子函数要非常高效地执行。如果空闲任务移植
    卡在钩子函数里的话，它就无法释放内存。



空闲任务的优先级为 0，这意味着一旦某个用户的任务变为就绪态，那么空闲任务马上被切换出去，让这个用户任务运行。在这种情况下，我们说用户任务"抢占"(pre-empt)了空闲任务，这是由调度器实现的。

要注意的是：如果使用 vTaskDelete()来删除任务，那么你就要确保空闲任务有机会执行，否则就无法释放被删除任务的内存。

### 2.使用钩子函数的前提：

在 FreeRTOS\Source\tasks.c 中，可以看到如下代码，所以前提就是：

-  	把这个宏定义为 1： configUSE_IDLE_HOOK
-  	实现 vApplicationIdleHook 函数

![image-20231204201100384](C:\Users\20215\AppData\Roaming\Typora\typora-user-images\image-20231204201100384.png)

## freertos之同步与互斥：

同步与互斥是两个非常重要的概念，接下来的队列、信号量、互斥量、事件组都会涉及到同步与互斥，所以很有必要在这里讲解一下。

**1.同步（Synchronization） 是指协调多个任务或线程的执行顺序和相互之间的行为，以确保它们按照一定的顺序、时机和约束进行执行。同步的目的是保证任务或线程之间的有序交互，使它们能够按照预期的顺序完成各自的操作或实现特定的约束条件。常见的同步场景包括等待其他任务完成、等待某个条件满足、协调任务之间的依赖关系等。**

**2.互斥（Mutual Exclusion） 是指限制共享资源在任何时刻只能被一个任务或线程访问的机制。当多个任务或线程需要访问共享资源时，互斥机制用于确保只有一个任务或线程可以进入临界区（即代码区段），并对共享资源进行操作。互斥的目的是避免竞争条件，保证共享资源的访问互不干扰，从而避免数据不一致性和冲突**
这样说可能有点难以理解，这里我们来举个例子：

假设有两个任务 TaskA 和 TaskB，它们需要访问共享资源（例如一个全局变量）并进行操作。

1. 同步的例子：
   假设 TaskA 和 TaskB **都需要按照特定的顺序执行**，而且它们共享一个标志变量 flag。具体场景如下：
   - TaskA 需要等待 flag 变为 true，然后才能执行特定的操作。
   - TaskB 在某个条件满足时将 flag 设置为 true，以通知 TaskA 可以执行操作。
   
2. 互斥的例子：
   假设 TaskA 和 TaskB **都需要访问和修改一个共享资源（例如一个全局变量）**，并且有可能发生数据竞争的情况。具体场景如下：
   - TaskA 和 TaskB 都需要读取和修改全局变量。（前面我们解释过一个变量如何进行读取和修改，这里就不深入解释了）
   - 如果两个任务同时访问和修改该变量，会导致不一致的结果。
   

## freeetos之队列的初步认识

### 1.队列的概念：

在FreeRTOS中，队列（Queue）是一种用于任务间通信和数据传递的机制。队列可以被看作是一个**先进先出**（FIFO）的缓冲区，任务可以向队列中发送数据项，也可以从队列中接收数据项。

### 2.队列的本质：环形缓冲区

环形缓冲区（Circular Buffer），也称为循环缓冲区或环形队列，是一种固定大小的缓冲区，其内部数据被组织成环形结构。它具有头指针和尾指针，并且可以循环利用存储空间。

环形缓冲区的特点是：
1. 固定大小：环形缓冲区在创建时需要指定其大小，即可以容纳的数据项数量。一旦创建后，大小通常是固定的，不会动态改变。
2. 环形结构：**环形缓冲区的内部数据项被组织成环形结构。当数据项被插入到尾部时，尾指针向前移动，并且当尾指针达到缓冲区的末尾时，会循环回到缓冲区的起始位置。同样，当数据项被取出时，头指针向前移动，并且当头指针达到缓冲区的末尾时，会循环回到缓冲区的起始位置。**
3. 循环利用：由于环形结构的特点，当尾指针达到缓冲区末尾时，新的数据项可以再次填充到缓冲区的起始位置上，达到循环利用存储空间的效果。

环形缓冲区常用于数据的生产者-消费者模型中，其中生产者将数据插入到缓冲区的尾部，而消费者从缓冲区的头部取出数据。通过使用环形缓冲区，可以实现高效的数据传输和存储，避免了数据的搬移操作，提高了数据的处理效率。

需要注意的是，当环形缓冲区已满时，新的数据插入会覆盖掉最早的数据；当环形缓冲区为空时，取出数据操作将无法执行。因此，在使用环形缓冲区时，需要合理设置缓冲区的大小，以满足实际需求，并避免数据丢失或阻塞的问题。

### 3.队列传输数据的方法：

在队列中传输数据，有两种常用的方法：值传递和指针传递。

1. 值传递：在值传递方式下，数据项被复制并传递给队列。当一个任务向队列发送数据时，数据项的副本被插入到队列中，而原始数据保持不变。当另一个任务从队列中接收数据时，接收到的是数据项的副本，而不是原始数据。

值传递的优点是简单、安全，因为每个任务都有自己的数据副本，不会出现数据竞争的问题。然而，当数据项较大或者需要频繁传输时，值传递会带来较大的开销，因为需要复制大量的数据。

2. 指针传递：在指针传递方式下，队列中存储的是指向数据的指针。当一个任务向队列发送数据时，实际的数据被保存在任务的内存空间中，而队列中存储的是指向该数据的指针。当另一个任务从队列中接收数据时，接收到的是指向数据的指针，通过该指针可以访问原始数据。

指针传递的优点是效率高，因为不需要复制大量的数据，只需要传递指针即可。然而，指针传递需要注意数据的有效性和共享的安全性，因为多个任务可能同时访问同一个数据对象。

根据实际需求和数据的特点，可以选择适合的传递方式。如果数据项较小且不频繁传输，值传递是一个简单有效的选择。如果数据项较大或需要频繁传输，指针传递可以提高效率。在使用指针传递时，需要确保数据的有效性和共享的安全性，可以通过使用互斥锁等机制进行保护。

### 4.队列的阻塞访问：

**队列的阻塞访问是指在队列操作中，当队列无法满足某个操作时，任务可以选择阻塞等待直到满足条件。**

只要知道队列的句柄，谁都可以读、写该队列。任务、 ISR 

都可读、写队列。可以多个任务读写队列。

任务读写队列时，简单地说：如果读写不成功，则阻塞；可以指定超时时间。口语化地说，就是可以定个闹钟：如果能读写了就马上进入就绪态，否则就阻塞直到超时。

某个任务读队列时，如果队列没有数据，则该任务可以进入阻塞状态：还可以指定阻塞的时间。如果队列有数据了，则该阻塞的任务会变为就绪态。如果一直都没有数据，则时间到之后它也会进入就绪态。

既然读取队列的任务个数没有限制，那么当多个任务读取空队列时，这些任务都会进入阻塞状态：有多个任务在等待同一个队列的数据。当队列中有数据时，哪个任务会进入就绪态？

- 优先级最高的任务

- 如果大家的优先级相同，那等待时间最久的任务会进入就绪态

跟读队列类似，一个任务要写队列时，如果队列满了，该任务也可以进入阻塞状态：还可以指定阻塞的时间。如果队列有空间了，则该阻塞的任务会变为就绪态。如果一直都没有空间，则时间到之后它也会进入就绪态。



既然写队列的任务个数没有限制，那么当多个任务写

"满队列"时，这些任务都会进入阻塞状态：有多个任务在等待同一个队列的空间。当队列中有空间时，哪个任务会进入就绪态？

- 优先级最高的任务
- 如果大家的优先级相同，那等待时间最久的任务会进入就绪态

### 5.队列函数：

（这里只介绍基本常用的函数，其他函数不过多介绍）

#### 1.队列创建函数：

```c
 QueueHandle_t xQueueCreate(
                             UBaseType_t  uxQueueLength,
                             UBaseType_t  uxItemSize
                         );
```



| 参数          | 说明                                                         |
| ------------- | ------------------------------------------------------------ |
| uxQueueLength | 队列长度，最多能存放多少个数据(item)                         |
| uxItemSize    | 每个数据(item)的大小：以字节为单位                           |
| 返回值        | 非 0 ： 成 功 ， 返 回 句 柄 ， 以 后 使 用 句 柄 来 操 作 队 列<br />NULL：失败，因为内存不足 |

#### 2.队列删除：

删除队列的函数为 vQueueDelete()，只能删除使用动态方法创建的队列，它会释放内存。 原型如下：

```c
void vQueueDelete( QueueHandle_t xQueue )
```



#### 3.队列的写入：

```c
/*
往队列尾部写入数据，如果没有空间，阻塞时间为 xTicksToWait
*/
BaseType_t xQueueSend(
QueueHandle_t xQueue,
const void *pvItemToQueue,
TickType_t xTicksToWait
);
```



| 参数          | 说明                                                         |
| ------------- | ------------------------------------------------------------ |
| xQueue        | 队列句柄，要写哪个队列                                       |
| pvItemToQueue | 数据指针，这个数据的值会被复制进队列，复制多大的数据？在创建队列时已经指定了数据大小 |
| xTicksToWait  | 如果队列满则无法写入新数据，可以让任务进入阻塞状态， xTicksToWait 表示阻塞的最大时间(Tick Count)。<br />如果被设为 0，无法写入数据时函数会立刻返回；<br />如果被设为 portMAX_DELAY，则会一直阻塞直到有空间可写 |
| 返回值        | pdPASS：数据成功写入了队列errQUEUE_FULL：写入失败，因为队列满了。 |

#### 4.队列的读取：

使用 xQueueReceive()函数读队列，读到一个数据后，队列中该数据会被移除。函数原型如下：

```c

BaseType_t xQueueReceive( QueueHandle_t xQueue,
void * const pvBuffer,
TickType_t xTicksToWait );
```



#### 5.示例：

```c
static int sum = 0;
static volatile int flagCalcEnd = 0;
static volatile int flagUARTused = 0;
static QueueHandle_t xQueueCalcHandle;
static QueueHandle_t xQueueUARTcHandle;

//创建队列
int InitUARTLock(void)
{	
	int val;
	xQueueUARTcHandle = xQueueCreate(1, sizeof(int));
	if (xQueueUARTcHandle == NULL)
	{
		printf("can not create queue\r\n");
		return -1;
	}
	xQueueSend(xQueueUARTcHandle, &val, portMAX_DELAY);
	return 0;
}

//定义了获取UART锁的函数GetUARTLock()，函数中通过接收队列的方式获取UART锁的值，并打印出来。
void GetUARTLock(void)
{	
	int val;
	xQueueReceive(xQueueUARTcHandle, &val, portMAX_DELAY);
	
}
//定义了释放UART锁的函数PutUARTLock()，函数中通过发送队列的方式释放UART锁，并打印释放的值。
void PutUARTLock(void)
{	
	int val;
	xQueueSend(xQueueUARTcHandle, &val, portMAX_DELAY);
	
}


void Task1Function(void * param)
{
	volatile int i = 0;
	while (1)
	{
		for (i = 0; i < 10000000; i++)
			sum++;
		
		//flagCalcEnd = 1;
		//vTaskDelete(NULL);
		xQueueSend(xQueueCalcHandle, &sum, portMAX_DELAY);
		printf("sum = %d\r\n", sum);
		sum = 1;

		
	}
}

void Task2Function(void * param)
{
	int val;
	
	while (1)
	{
		//if (flagCalcEnd)
		flagCalcEnd = 0;
		xQueueReceive(xQueueCalcHandle, &val, portMAX_DELAY);
		flagCalcEnd = 1;
		

		
	}
}

void TaskGenericFunction(void * param)
{
	while (1)
	{
		GetUARTLock();
		printf("%s\r\n", (char *)param);
		PutUARTLock(); /* task 3 ==> ready, task 4 is running  */
		vTaskDelay(1);
	}
}


/*-----------------------------------------------------------*/

int main( void )
{
	TaskHandle_t xHandleTask1;
		
#ifdef DEBUG
  debug();
#endif

	prvSetupHardware();

	printf("Hello, world!\r\n");

	xQueueCalcHandle = xQueueCreate(10, sizeof(int));
	if (xQueueCalcHandle == NULL)
	{
		printf("can not create queue\r\n");
	}

	InitUARTLock();

//	xTaskCreate(Task1Function, "Task1", 100, NULL, 1, &xHandleTask1);
//	xTaskCreate(Task2Function, "Task2", 100, NULL, 1, NULL);

	xTaskCreate(TaskGenericFunction, "Task3", 100, "Task 3 is running", 1, NULL);
	xTaskCreate(TaskGenericFunction, "Task4", 100, "Task 4 is running", 1, NULL);

	/* Start the scheduler. */
	vTaskStartScheduler();

	/* Will only get here if there was not enough heap space to create the
	idle task. */
	return 0;
}
```

通过串口打印可以看到

![image-20231213145009489](C:\Users\20215\AppData\Roaming\Typora\typora-user-images\image-20231213145009489.png)



![image-20231213145308223](C:\Users\20215\AppData\Roaming\Typora\typora-user-images\image-20231213145308223.png)

很明显队列的使用可以实现高效的数据传输和存储。

## freertos之信号量：

### 1.信号量的概念：

**信号量是一种计数器**，用于控制对共享资源的访问。它可以用来解决多个线程之间的竞争条件和同步问题。信号量维护着一个**非负整数**值，表示可用资源的数量。线程可以通过获取（取得）或释放（归还）信号量来进行资源的获取和释放。

这个可以举一个简单的例子：

假设有一个简单的场景：一个停车场，停车位有限，只能容纳3辆车。多个车辆同时到达停车场，并且需要等待空闲的停车位才能停车。

在这个场景中，可以使用信号量来控制对停车位的访问。以下是一个生动形象的说明：

1. 停车场初始化时创建一个计数型信号量，初始计数器值为3，表示有3个可用的停车位。
2. 当一辆车到达停车场时，车辆需要获取信号量。如果信号量的计数器大于0，表示有可用的停车位，车辆可以顺利停车，计数器减1。如果计数器为0，表示所有停车位都被占用，车辆需要等待（即进入阻塞态）。
3. 当一辆车停车完成准备离开时，它会释放信号量，计数器加1。此时，其他等待的车辆可以获取信号量，顺利停车。
4. 如果有更多的车辆到达停车场，并且所有停车位都被占用，它们仍然需要等待，直到有车辆离开并释放信号量。很明显如果有很多量车进行等待那么，按我们的日常习惯来说，如果大家都没有什么非常紧急的事情（即优先级相同）下一个停车位就轮到那个先来等车的人（等待时间最长的人），如果有急事的话可以允许你先占用这个停车位（高优先级先进行）

通过使用信号量，可以确保停车场中同时停车的车辆不超过3辆，避免超过停车位容量，保持停车场的有序管理。

这个例子中的信号量就像是一个计数器，表示可用的停车位数量。车辆在获取信号量之前必须等待，直到有可用的停车位，而离开时会释放信号量，增加可用停车位的计数器值。这种机制实现了对停车位的访问控制和资源管理。



### 2.计数型信号量：

计数信号量可以具有大于 1 的计数器值，表示可用资源的数量。它通常用于控制对一组资源的访问，例如有限数量的缓冲区或资源池。线程可以获取计数信号量来获取资源，并在使用完资源后释放信号量，增加计数器的值。当计数器达到最大值时，其他线程会被阻塞，直到有线程释放信号量。

### 3.二进制信号量：

二进制信号量只有两种状态：可用和不可用。它通常用于实现互斥锁（Mutex）的功能，用于对共享资源的互斥访问。当一个线程获取二进制信号量时，如果信号量可用，则减少计数器并继续执行；如果信号量不可用，则线程会被阻塞，直到其他线程释放信号量。

### 4.计数型信号量和二进制信号量的区别：

- 计数型信号量：计数型信号量的计数器可以具有大于1的值，表示可用资源的数量。线程可以根据需要获取多个资源，每个获取操作会减少计数器的值，释放操作会增加计数器的值。
- 二进制信号量：二进制信号量的计数器值只能是0或1，表示资源的占用状态。线程可以获取（将计数器设置为1）或释放（将计数器设置为0）该信号量，用于实现互斥访问。



### 5.信号量相关函数：

#### 1.计数型信号量创建函数：

```c
/* 
创建一个计数型信号量，返回它的句柄。
* 此函数内部会分配信号量结构体
* uxMaxCount: 最大计数值
* uxInitialCount: 初始计数值
* 返回值: 返回句柄，非 NULL 表示成功
*/
SemaphoreHandle_t xSemaphoreCreateCounting(UBaseType_t uxMaxCount, UBaseType_t ux
InitialCount);
```

#### 2.二进制信号量创建函数：

```c
/* 
创建一个二进制信号量，返回它的句柄。
* 此函数内部会分配信号量结构体
* 返回值: 返回句柄，非 NULL 表示成功
*/
SemaphoreHandle_t xSemaphoreCreateBinary( void );
```

#### 3.信号量删除函数：

计数型信号量和二进制信号量的删除函数是一样的

```c
/*
* xSemaphore: 信号量句柄，你要删除哪个信号量
*/
void vSemaphoreDelete( SemaphoreHandle_t xSemaphore );
```

#### 4.信号量操作函数：

计数型信号量和二进制信号量的操作函数是一样的

```c
/*
xSemaphoreGive操纵函数声明
*xSemaphore：信号量句柄，释放那个信号量
*返回值：
        pdTRUE 表示成功,
        如果二进制信号量的计数值已经是 1，再次调用此函数则返回失败；
        如果计数型信号量的计数值已经是最大值，再次调用此函数则返回失败
*/
BaseType_t xSemaphoreGive( SemaphoreHandle_t xSemaphore );
/*
xSemaphoreTake操作函数声明
*xSemaphore：信号量句柄，获取那个信号量
*xTicksToWait：如果无法马上获得信号量，阻塞一会：
                                0：不阻塞，马上返回
                                portMAX_DELAY: 一直阻塞直到成功
                                其他值: 阻塞的 Tick 个数，可以使用 pdMS_TO_TICKS()来指定阻塞时间为若干 ms
*返回值：pdTRUE 表示成功。                               
*/
BaseType_t xSemaphoreTake(
 SemaphoreHandle_t xSemaphore,
 TickType_t xTicksToWait
 );
```





### 5.信号量的应用：

```c
static int sum = 0;
static volatile int flagCalcEnd = 0;
static volatile int flagUARTused = 0;

static SemaphoreHandle_t xSemCalc;
static SemaphoreHandle_t xSemUART;


void Task1Function(void * param)
{
	volatile int i = 0;
	while (1)
	{
		for (i = 0; i < 10000000; i++)
			sum++;
		//printf("1");
		xSemaphoreGive(xSemCalc);
		
		vTaskDelete(NULL);
	}
}

void Task2Function(void * param)
{
	while (1)
	{
		//if (flagCalcEnd)
		flagCalcEnd = 0;
		xSemaphoreTake(xSemCalc, portMAX_DELAY);
		flagCalcEnd = 1;
		printf("sum = %d\r\n", sum);
	}
}

void TaskGenericFunction(void * param)
{
	while (1)
	{
		xSemaphoreTake(xSemUART, portMAX_DELAY);
		printf("%s\r\n", (char *)param);
		xSemaphoreGive(xSemUART);
		vTaskDelay(1);
	}
}


/*-----------------------------------------------------------*/

int main( void )
{
	TaskHandle_t xHandleTask1;
		
#ifdef DEBUG
  debug();
#endif

	prvSetupHardware();

	printf("Hello, world!\r\n");
	xSemCalc = xSemaphoreCreateCounting(10, 0);//创建计数型信号量 最大计数值  初始计数值
	xSemUART = xSemaphoreCreateBinary(); // 二进制信号量 
	xSemaphoreGive(xSemUART);

	xTaskCreate(Task1Function, "Task1", 100, NULL, 1, &xHandleTask1);
	xTaskCreate(Task2Function, "Task2", 100, NULL, 1, NULL);

	//xTaskCreate(TaskGenericFunction, "Task3", 100, "Task 3 is running", 1, NULL);
	//xTaskCreate(TaskGenericFunction, "Task4", 100, "Task 4 is running", 1, NULL);

	/* Start the scheduler. */
	vTaskStartScheduler();

	/* Will only get here if there was not enough heap space to create the
	idle task. */
	return 0;
}
```

计数型信号量实验结果：

![image-20231215210013487](C:\Users\20215\AppData\Roaming\Typora\typora-user-images\image-20231215210013487.png)

二进制信号量实验结果：

![image-20231215210507209](C:\Users\20215\AppData\Roaming\Typora\typora-user-images\image-20231215210507209.png)

因为这个实验实现的功能与队列哪里实现的功能是一样所以不详细解释了



### 6.信号量和队列的区别：

下面是一个表格，说明了信号量和队列之间的一些区别：

| 特征         | 信号量                             | 队列                                 |
| ------------ | ---------------------------------- | ------------------------------------ |
| 主要功能     | 控制对共享资源的访问               | 管理和传递数据                       |
| 操作方式     | 获取和释放信号量                   | 入队和出队                           |
| 计数器       | 用于表示可用资源的数量             | 无计数器，按照先进先出顺序管理元素   |
| 初始值       | 可以为任意大于等于0的整数          | 可以为空或包含初始元素               |
| 值的范围     | 可以是大于1的整数或0               | 可以是任意类型的元素                 |
| 同步行为     | 当计数为0时，线程等待获取信号量    | 队列为空时，出队操作会等待元素的到来 |
| 阻塞行为     | 线程在获取信号量时可能被阻塞或等待 | 出队操作在队列为空时可能被阻塞或等待 |
| 用途         | 控制资源的访问、并发处理和资源管理 | 多任务协调、消息传递和事件处理       |
| 数据结构类型 | 无具体数据结构要求，可用于各种场景 | 基于先进先出（FIFO）的数据结构       |



## freertos之互斥量：

### 1.互斥量的概念：

FreeRTOS 中的互斥量（Mutex）是一种同步机制，用于实现任务之间对共享资源的互斥访问。互斥量的概念基于二进制信号量（Binary Semaphore），它允许只有一个任务同时获取互斥量的所有权。

互斥量的核心思想是，只有一个任务可以拥有互斥量的所有权，而其他任务需要等待该互斥量被释放。这种机制确保了在任意时刻只有一个任务能够访问共享资源，从而避免了数据竞争和不一致性的问题。

举个例子：

怎么独享厕所？**自己开门上锁，完事了自己开锁**。
 你当然可以进去后，让别人帮你把门：但是，命运就掌握在别人手上了。
 使用队列、信号量，都可以实现互斥访问，以信号量为例：
 	信号量初始值为 1
	任务 A 想上厕所，"take"信号量成功，它进入厕所
	 任务 B 也想上厕所，"take"信号量不成功，等待
	 任务 A 用完厕所，"give"信号量；轮到任务 B 使用
 这需要有 2 个前提：
	任务 B 很老实，不撬门(一开始不"give"信号量)
	没有坏人：别的任务不会"give"信号量
	 可以看到，使用信号量确实也可以实现互斥访问，但是不完美。
 使用互斥量可以解决这个问题，互斥量的名字取得很好：
 	**量：值为 0、1**
 	**互斥：用来实现互斥访问**
 它的核心在于：**谁上锁，就只能由谁开锁**。
 很奇怪的是，FreeRTOS 的互斥锁，并没有在代码上实现这点：

​	即使任务 A 获得了互斥锁，任务 B 竟然也可以释放互斥锁。
​	谁上锁、谁释放：只是约定

而rtthread则实现了谁上锁，就只能谁来开锁的这一功能。

### 2.互斥量的使用场合：

在多任务系统中，任务 A 正在使用某个资源，还没用完的情况下任务 B 也来使用的话，

就可能导致问题。比如对于串口，任务 A 正使用它来打印，在打印过程中任务 B 也来打印，客户看到的

结果就是 A、B 的信息混杂在一起。

这种现象很常见：

- 访问外设：刚举的串口例子
- 读、修改、写操作导致的问题

对于同一个变量，比如 int a，如果有两个任务同时写它就有可能导致问题。

对于变量的修改，C 代码只有一条语句，比如：a=a+8;，它的内部实现分为 3 步：读出原

值、修改、写入。

![image-20231216192144417](C:\Users\20215\AppData\Roaming\Typora\typora-user-images\image-20231216192144417.png)

我们想让任务 A 、 B 都执行 add_a 函数， a 的最终结果是 1+8+8=17 。

假设任务 A 运行完代码①，在执行代码②之前被任务 B 抢占了：现在任务 A 的 R0 等于 1。

任务 B 执行完 add_a 函数，a 等于 9。任务 A 继续运行，在代码②处 R0 仍然是被抢占前的数值 1，执行完②③的代码，a

等于 9，这跟预期的 17 不符合。

- 对变量的非原子化访问

​		修改变量、设置结构体、在 16 位的机器上写 32 位的变量，这些操作都是非原子的。也就是它们的操作过程都可能被打断，如果被		打断的过程有其他任务来操作这些变量，就可能导致冲突。

- 函数重入

​		"可重入的函数"是指：多个任务同时调用它、任务和中断同时调用它，函数的运行也是安全的。**可重入的函数也被称为"线程安全"		(thread safe)。**每个任务都维持自己的栈、自己的 CPU 寄存器，如果一个函数只使用局部变量，那么它就是线程安全的。

​		函数中一旦使用了全局变量、静态变量、其他外设，它就不是"可重入的"，如果该函数正在被调用，就必须阻止其他任务、中断再次		调用它。

上述问题的解决方法是：任务 A 访问这些全局变量、函数代码时，独占它，就是上个

锁。这些全局变量、函数代码必须被独占地使用，它们被称为**临界资源**。

互斥量也被称为互斥锁，使用过程如下：

- 互斥量初始值为 1
- 任务 A 想访问临界资源，先获得并占有互斥量，然后开始访问
- 任务 B 也想访问临界资源，也要先获得互斥量：被别人占有了，于是阻塞
- 任务 A 使用完毕，释放互斥量；任务 B 被唤醒、得到并占有互斥量，然后
- 开始访问临界资源
- 任务 B 使用完毕，释放互斥量

**正常来说：在任务 A 占有互斥量的过程中，任务 B、任务 C 等等，都无法释放互斥量。**

**但是 FreeRTOS 未实现这点：任务 A 占有互斥量的情况下，任务 B 也可释放互斥量。**

### 3.互斥量相关函数：

#### 1.互斥量创建函数：

```c
/* 
创建一个互斥量，返回它的句柄。
* 此函数内部会分配互斥量结构体
* 返回值: 返回句柄，非 NULL 表示成功
*/
SemaphoreHandle_t xSemaphoreCreateMutex( void );
```

要想使用互斥量，需要在配置文件 FreeRTOSConfig.h 中定义：

```c
define configUSE_MUTEXES 1
```

#### 2.其他函数：

要注意的是，互斥量不能在 ISR 中使用。

各类操作函数，比如删除、give/take，跟一般是信号量是一样的。所以这里就不详细解释函数参数了。

```c
/*
* xSemaphore: 信号量句柄，你要删除哪个信号量, 互斥量也是一种信号量
*/
void vSemaphoreDelete( SemaphoreHandle_t xSemaphore );
/* 释放 */
BaseType_t xSemaphoreGive( SemaphoreHandle_t xSemaphore );
/* 获得 */
BaseType_t xSemaphoreTake(
 SemaphoreHandle_t xSemaphore,
 TickType_t xTicksToWait
 );
```

#### 3.示例：

用互斥量访问串口

```c
extern void vSetupTimerTest( void );

/*-----------------------------------------------------------*/

static int sum = 0;
static volatile int flagCalcEnd = 0;
static volatile int flagUARTused = 0;

static SemaphoreHandle_t xSemCalc;
static SemaphoreHandle_t xSemUART;


void Task1Function(void * param)
{
	volatile int i = 0;
	while (1)
	{
		for (i = 0; i < 10000000; i++)
			sum++;
		//printf("1");
		xSemaphoreGive(xSemCalc);
		vTaskDelete(NULL);
	}
}

void Task2Function(void * param)
{
	while (1)
	{
		//if (flagCalcEnd)
		flagCalcEnd = 0;
		xSemaphoreTake(xSemCalc, portMAX_DELAY);
		flagCalcEnd = 1;
		printf("sum = %d\r\n", sum);
	}
}

void TaskGenericFunction(void * param)
{
	while (1)
	{
		xSemaphoreTake(xSemUART, portMAX_DELAY);
		printf("%s\r\n", (char *)param);
		xSemaphoreGive(xSemUART);
		vTaskDelay(1);
	}
}


/*-----------------------------------------------------------*/

int main( void )
{
	TaskHandle_t xHandleTask1;
		
#ifdef DEBUG
  debug();
#endif

	prvSetupHardware();

	printf("Hello, world!\r\n");
	xSemCalc = xSemaphoreCreateCounting(10, 0);
	//xSemUART = xSemaphoreCreateBinary();
	//xSemaphoreGive(xSemUART);
	
	xSemUART = xSemaphoreCreateMutex();

//	xTaskCreate(Task1Function, "Task1", 100, NULL, 1, &xHandleTask1);
//	xTaskCreate(Task2Function, "Task2", 100, NULL, 1, NULL);

	xTaskCreate(TaskGenericFunction, "Task3", 100, "Task 3 is running", 1, NULL);
	xTaskCreate(TaskGenericFunction, "Task4", 100, "Task 4 is running", 1, NULL);

	/* Start the scheduler. */
	vTaskStartScheduler();

	/* Will only get here if there was not enough heap space to create the
	idle task. */
	return 0;
}
```

结果如图显示：



![image-20231216200539401](C:\Users\20215\AppData\Roaming\Typora\typora-user-images\image-20231216200539401.png)

### 4.**优先级翻转**：

二进制信号量的缺陷。有 3 个任务，优先级分别为 H，M，L（即高中低）。L任务负责释放互斥量，由于未能成功释放，导致被 H 任务抢占，该任务需要获取互斥量，但是 L 任务未成功释放这时由于任务的特性 H 任务会进入阻塞态（这里产生了优先级翻转）。由于 H 任务处于阻塞态 L 任务被内核调度，在运行的这段期间又被 M 任务抢占，这时 L 任务还没有释放互斥量又进入了阻塞态，内核这时去调度 M 任务，执行完后在调度 L 任务，直到 L 任务释放互斥量 H 任务才能得到 内核的调度。从中可以发现原本 H 任务的优先级是最高的，反而最后执行，这样的现象称为 “优先级翻转”。

在RTOS中优先级翻转是致命的，它是造成系统崩溃的原凶，它会打断系统任务规定的执行顺序。在 FreeRTOS 中优先级翻转是无法根治的，只能把危害降到最低，这就要引入互斥量了，互斥量有个特性，优先级继承。

案例：

```c
static volatile uint8_t flagLPTaskRun = 0;
static volatile uint8_t flagMPTaskRun = 0;
static volatile uint8_t flagHPTaskRun = 0;

static void vLPTask( void *pvParameters );
static void vMPTask( void *pvParameters );
static void vHPTask( void *pvParameters );

/*-----------------------------------------------------------*/

/* 互斥量/二进制信号量句柄 */
SemaphoreHandle_t xLock;

int main( void )
{
	prvSetupHardware();
	
    /* 创建互斥量/二进制信号量 */
    xLock = xSemaphoreCreateBinary( );
	xSemaphoreGive(xLock);


	if( xLock != NULL )
	{
		/* 创建3个任务: LP,MP,HP(低/中/高优先级任务)
		 */
		xTaskCreate( vLPTask, "LPTask", 1000, NULL, 1, NULL );
		xTaskCreate( vMPTask, "MPTask", 1000, NULL, 2, NULL );
		xTaskCreate( vHPTask, "HPTask", 1000, NULL, 3, NULL );

		/* 启动调度器 */
		vTaskStartScheduler();
	}
	else
	{
		/* 无法创建互斥量/二进制信号量 */
	}

	/* 如果程序运行到了这里就表示出错了, 一般是内存不足 */
	return 0;
}

/*-----------------------------------------------------------*/

/*-----------------------------------------------------------*/
static void vLPTask( void *pvParameters )
{
	const TickType_t xTicksToWait = pdMS_TO_TICKS( 10UL );	
	uint32_t i;
	char c = 'A';

	printf("LPTask start\r\n");
	
	/* 无限循环 */
	for( ;; )
	{	
		flagLPTaskRun = 1;
		flagMPTaskRun = 0;
		flagHPTaskRun = 0;

		/* 获得互斥量/二进制信号量 */
		xSemaphoreTake(xLock, portMAX_DELAY);
		
		/* 耗时很久 */
		
		printf("LPTask take the Lock for long time");
		for (i = 0; i < 500; i++) 
		{
			flagLPTaskRun = 1;
			flagMPTaskRun = 0;
			flagHPTaskRun = 0;
			printf("%c", c + i);
		}
		printf("\r\n");
		
		/* 释放互斥量/二进制信号量 */
		xSemaphoreGive(xLock);
		
		vTaskDelay(xTicksToWait);
	}
}

static void vMPTask( void *pvParameters )
{
	const TickType_t xTicksToWait = pdMS_TO_TICKS( 30UL );	

	flagLPTaskRun = 0;
	flagMPTaskRun = 1;
	flagHPTaskRun = 0;

	printf("MPTask start\r\n");
	
	/* 让LPTask、HPTask先运行 */	
	vTaskDelay(xTicksToWait);
	
	/* 无限循环 */
	for( ;; )
	{	
		flagLPTaskRun = 0;
		flagMPTaskRun = 1;
		flagHPTaskRun = 0;
	}
}

static void vHPTask( void *pvParameters )
{
	const TickType_t xTicksToWait = pdMS_TO_TICKS( 10UL );	

	flagLPTaskRun = 0;
	flagMPTaskRun = 0;
	flagHPTaskRun = 1;

	printf("HPTask start\r\n");
	
	/* 让LPTask先运行 */	
	vTaskDelay(xTicksToWait);
	
	/* 无限循环 */
	for( ;; )
	{	
		flagLPTaskRun = 0;
		flagMPTaskRun = 0;
		flagHPTaskRun = 1;
		printf("HPTask wait for Lock\r\n");
		
		/* 获得互斥量/二进制信号量 */
		xSemaphoreTake(xLock, portMAX_DELAY);
		
		flagLPTaskRun = 0;
		flagMPTaskRun = 0;
		flagHPTaskRun = 1;
		
		/* 释放互斥量/二进制信号量 */
		xSemaphoreGive(xLock);
	}
}

/*-----------------------------------------------------------*/


/*-----------------------------------------------------------*/

```



结果如图所示：

![qq_pic_merged_1702733597579](D:\QMDownload\MobileFile\qq_pic_merged_1702733597579.jpg)

### 5.**优先级继承**：

分析环境同上，一样是 3 个任务 H，M，L。L 任务还是负责释放信号量，H任务抢占了 L 的内核使用权，要获取信号量，这时由于 L 任务还未能成功释放信号量，H 进入阻塞，这时 L 任务会把自己的优先级设置成与 H 任务的优先级一致，这种现象称为“优先级继承”。由于 L 任务的优先级与 H 任务的一致，M 任务就无法打断了。L 任务继续运行，直到释放了信号量，优先级才会恢复到原来的状态，由于 L 任务释放了信号量，H 任务得到执行，期间 M 任务无法抢占 L，在 L 释放信号量并恢复优先级后，任务调度的顺序就确定了，即 H，M，L。

案例：

```c
static volatile uint8_t flagLPTaskRun = 0;
static volatile uint8_t flagMPTaskRun = 0;
static volatile uint8_t flagHPTaskRun = 0;

static void vLPTask( void *pvParameters );
static void vMPTask( void *pvParameters );
static void vHPTask( void *pvParameters );

/*-----------------------------------------------------------*/

/* 互斥量/二进制信号量句柄 */
SemaphoreHandle_t xLock;

int main( void )
{
	prvSetupHardware();
	
    /* 创建互斥量/二进制信号量 */
    //xLock = xSemaphoreCreateBinary( );
	//xSemaphoreGive(xLock);
	xLock = xSemaphoreCreateMutex( );


	if( xLock != NULL )
	{
		/* 创建3个任务: LP,MP,HP(低/中/高优先级任务)
		 */
		xTaskCreate( vLPTask, "LPTask", 1000, NULL, 1, NULL );
		xTaskCreate( vMPTask, "MPTask", 1000, NULL, 2, NULL );
		xTaskCreate( vHPTask, "HPTask", 1000, NULL, 3, NULL );

		/* 启动调度器 */
		vTaskStartScheduler();
	}
	else
	{
		/* 无法创建互斥量/二进制信号量 */
	}

	/* 如果程序运行到了这里就表示出错了, 一般是内存不足 */
	return 0;
}

/*-----------------------------------------------------------*/

/*-----------------------------------------------------------*/
static void vLPTask( void *pvParameters )
{
	const TickType_t xTicksToWait = pdMS_TO_TICKS( 10UL );	
	uint32_t i;
	char c = 'A';

	printf("LPTask start\r\n");
	
	/* 无限循环 */
	for( ;; )
	{	
		flagLPTaskRun = 1;
		flagMPTaskRun = 0;
		flagHPTaskRun = 0;

		/* 获得互斥量/二进制信号量 */
		xSemaphoreTake(xLock, portMAX_DELAY);
		
		/* 耗时很久 */
		
		printf("LPTask take the Lock for long time");
		for (i = 0; i < 500; i++) 
		{
			flagLPTaskRun = 1;
			flagMPTaskRun = 0;
			flagHPTaskRun = 0;
			printf("%c", c + i);
		}
		printf("\r\n");
		
		/* 释放互斥量/二进制信号量 */
		xSemaphoreGive(xLock);
		
		vTaskDelay(xTicksToWait);
	}
}

static void vMPTask( void *pvParameters )
{
	const TickType_t xTicksToWait = pdMS_TO_TICKS( 30UL );	

	flagLPTaskRun = 0;
	flagMPTaskRun = 1;
	flagHPTaskRun = 0;

	printf("MPTask start\r\n");
	
	/* 让LPTask、HPTask先运行 */	
	vTaskDelay(xTicksToWait);
	
	/* 无限循环 */
	for( ;; )
	{	
		flagLPTaskRun = 0;
		flagMPTaskRun = 1;
		flagHPTaskRun = 0;
	}
}

static void vHPTask( void *pvParameters )
{
	const TickType_t xTicksToWait = pdMS_TO_TICKS( 10UL );	

	flagLPTaskRun = 0;
	flagMPTaskRun = 0;
	flagHPTaskRun = 1;

	printf("HPTask start\r\n");
	
	/* 让LPTask先运行 */	
	vTaskDelay(xTicksToWait);
	
	/* 无限循环 */
	for( ;; )
	{	
		flagLPTaskRun = 0;
		flagMPTaskRun = 0;
		flagHPTaskRun = 1;
		printf("HPTask wait for Lock\r\n");
		
		/* 获得互斥量/二进制信号量 */
		xSemaphoreTake(xLock, portMAX_DELAY);
		
		flagLPTaskRun = 0;
		flagMPTaskRun = 0;
		flagHPTaskRun = 1;
		
		/* 释放互斥量/二进制信号量 */
		xSemaphoreGive(xLock);
	}
}

```

结果如图所示：

![qq_pic_merged_1702733523349](D:\QMDownload\MobileFile\qq_pic_merged_1702733523349.jpg)

**注意**：优先级继承机制，只对低优先级任务有效，即 L 任务释放互斥量，被 H 任务获取期间 L 任务会进行优先级继承，如果一个比 L 任务优先级低的任务获取，这时 L 任务不会进行优先级继承。



### 6.递归锁：

#### **1.** 死锁的概念：

日常生活的死锁：我们只招有工作经验的人！我没有工作经验怎么办？那你就去找工作啊！

假设有 2 个互斥量 M1、M2，2 个任务 A、B：

- A 获得了互斥量 M1
-  B 获得了互斥量 M2
-  A 还要获得互斥量 M2 才能运行，结果 A 阻塞
- B 还要获得互斥量 M1 才能运行，结果 B 阻塞
- A、B 都阻塞，再无法释放它们持有的互斥量
- 死锁发生！

#### 2.自我死锁：

假设这样的场景：

- 任务 A 获得了互斥锁 M
- 它调用一个库函数
- 库函数要去获取同一个互斥锁 M，于是它阻塞：任务 A 休眠，等待任务 A来释放互斥锁！
- 死锁发生！

#### 3.相关函数：

怎么解决这类问题？可以使用递归锁(Recursive Mutexes)，它的特性如下：

- 任务 A 获得递归锁 M 后，它还可以多次去获得这个锁
- "take"了 N 次，要"give"N 次，这个锁才会被释放

递归锁的函数根一般互斥量的函数名不一样，参数类型一样，列表如下：

|      | 递归锁                         | 一般互斥量            |
| ---- | ------------------------------ | --------------------- |
| 创建 | xSemaphoreCreateRecursiveMutex | xSemaphoreCreateMutex |
| 获得 | xSemaphoreTakeRecursive        | xSemaphoreTake        |
| 释放 | xSemaphoreGiveRecursive        | xSemaphoreGive        |



```c
/* 创建一个递归锁，返回它的句柄。
* 此函数内部会分配互斥量结构体
* 返回值: 返回句柄，非 NULL 表示成功
*/
SemaphoreHandle_t xSemaphoreCreateRecursiveMutex( void );
/* 释放 */
BaseType_t xSemaphoreGiveRecursive( SemaphoreHandle_t xSemaphore );
/* 获得 */
BaseType_t xSemaphoreTakeRecursive(
 SemaphoreHandle_t xSemaphore,
 TickType_t xTicksToWait
 );
```



#### 4.示例：

```c
static int sum = 0;
static volatile int flagCalcEnd = 0;
static volatile int flagUARTused = 0;

static SemaphoreHandle_t xSemCalc;
static SemaphoreHandle_t xSemUART;


void Task1Function(void * param)
{
	volatile int i = 0;
	while (1)
	{
		for (i = 0; i < 10000000; i++)
			sum++;
		//printf("1");
		xSemaphoreGive(xSemCalc);
		vTaskDelete(NULL);
	}
}

void Task2Function(void * param)
{
	while (1)
	{
		//if (flagCalcEnd)
		flagCalcEnd = 0;
		xSemaphoreTake(xSemCalc, portMAX_DELAY);
		flagCalcEnd = 1;
		printf("sum = %d\r\n", sum);
	}
}

void TaskGenericFunction(void * param)
{
	int i;
	while (1)
	{
		xSemaphoreTakeRecursive(xSemUART, portMAX_DELAY);

		printf("%s\r\n", (char *)param);
		for (i = 0; i < 10; i++)
		{
			xSemaphoreTakeRecursive(xSemUART, portMAX_DELAY);
			printf("%s in loop %d\r\n", (char *)param, i);
			xSemaphoreGiveRecursive(xSemUART);
		}
		
		xSemaphoreGiveRecursive(xSemUART);
		vTaskDelay(1);
	}
}

void Task5Function(void * param)
{
	vTaskDelay(10);
	while (1)
	{
		while (1)
		{
			if (xSemaphoreTakeRecursive(xSemUART, 0) != pdTRUE)
			{
				xSemaphoreGiveRecursive(xSemUART);			
			}
			else
			{
				break;
			}
		}
		printf("%s\r\n", (char *)param);
		xSemaphoreGiveRecursive(xSemUART);
		vTaskDelay(1);
	}
}


/*-----------------------------------------------------------*/

int main( void )
{
	TaskHandle_t xHandleTask1;
		
#ifdef DEBUG
  debug();
#endif

	prvSetupHardware();

	printf("Hello, world!\r\n");
	xSemCalc = xSemaphoreCreateCounting(10, 0);
	//xSemUART = xSemaphoreCreateBinary();
	//xSemaphoreGive(xSemUART);
	
	//xSemUART = xSemaphoreCreateMutex();
	xSemUART = xSemaphoreCreateRecursiveMutex();

	xTaskCreate(Task1Function, "Task1", 100, NULL, 1, &xHandleTask1);
	xTaskCreate(Task2Function, "Task2", 100, NULL, 1, NULL);

	xTaskCreate(TaskGenericFunction, "Task3", 100, "Task 3 is running", 1, NULL);
	xTaskCreate(TaskGenericFunction, "Task4", 100, "Task 4 is running", 1, NULL);
	xTaskCreate(Task5Function, "Task5", 100, "Task 5 is running", 1, NULL);

	/* Start the scheduler. */
	vTaskStartScheduler();

	/* Will only get here if there was not enough heap space to create the
	idle task. */
	return 0;
}
/*-----------------------------------------------------------*/

/*-----------------------------------------------------------*/

/*-----------------------------------------------------------*/
```































创建队列的时候要指定长度、数据大小

数据的操作采用先进先出的方法，写数据的时候放到尾部，读数据的时候从头部开始读

队列的本质是环形缓冲区

创建队列的时候两个指针指向头部 

队列集









任务挂起函数

void vTaskSuspend(TaskHandle_t xTaskToSuspend) 

函数用于挂起任务，需要将宏INCLUDE_vTaskSuspend 置为1

无论优先级如何挂起的都任务不会被执行，直到任务被恢复

当函数的参数为NULL，则代码挂起当前正在运行的任务

![image-20231016082811337](C:\Users\20215\AppData\Roaming\Typora\typora-user-images\image-20231016082811337.png)















任务恢复被挂起的函数

void vTaskResume(TaskHandle_t xTaskToResume) 

函数用于挂起任务，需要将宏INCLUDE_vTaskSuspend置为1

调用该函数被挂起的任务就会继续运行进入就绪态

![image-20231016083315349](C:\Users\20215\AppData\Roaming\Typora\typora-user-images\image-20231016083315349.png)



中断中恢复被挂起的函数

BaseType_t xTaskResumeFromISR(TaskHandle_t xTaskToResume)

调用该函数的时候需要知道其返回值

如果是pdTRUE，则任务恢复后需要进行任务切换

如果是pdFALSE，则任务恢复后不需要进行任务切换

使用该函数注意宏：INCLUDE_vTaskSuspend 和 INCLUDE_xTaskResumeFromISR 必须定义为 1

注意：中断服务函数程序中如果要调用freertos的API函数则中断优先级不能高于freertos所管理的最高优先级



![image-20231016084058105](C:\Users\20215\AppData\Roaming\Typora\typora-user-images\image-20231016084058105.png)

### 2.TCB结构体的介绍

在 FreeRTOS 中，TCB（Task Control Block）是用于管理任务的结构体。**每个任务都有一个对应的 TCB 结构体，用于存储任务的信息和状态**。TCB 结构体的定义可以根据使用的具体 FreeRTOS 版本和配置而有所不同，下面是一个典型的 TCB 结构体的示例：

```c
c复制代
```

```c
typedef struct tskTaskControlBlock
{
    volatile StackType_t *pxTopOfStack;     // 指向任务堆栈顶部的指针
    ListItem_t xStateListItem;               // 用于在就绪列表和等待列表中连接任务的链表项
    ListItem_t xEventListItem;               // 用于在事件列表中连接任务的链表项
    UBaseType_t uxPriority;                   // 任务的优先级
    StackType_t *pxStack;                       // 指向任务堆栈基址的指针
    char pcTaskName[configMAX_TASK_NAME_LEN];    // 任务名称
    ...
    // 其他任务相关的字段和信息
} tskTCB;
```

  1. `pxTopOfStack`：这是一个指向任务堆栈顶部的指针，用于存储任务的堆栈顶部地址。

  2. `xStateListItem`：该成员是一个链表项，用于将任务连接到就绪列表和等待列表中。通过调整链表项的位置，可以使任务从一个列表移动到另一个列表中，从而实现任务的调度和等待。

  3. xEventListItem`：类似于 `xStateListItem`，`xEventListItem` 是一个链表项，用于将任务连接到事件列表中。任务可以等待某个事件的发生，当事件发生时，任务将被移动到就绪列表中。 

  4. uxPriority`：该成员存储任务的优先级。优先级值越高，任务获得的执行时间越多，优先级值越低，任务获得的执行时间越少。 `

  5.  pxStack`：指向任务堆栈基址的指针。任务堆栈是用来保存任务执行时的上下文信息，包括函数调用的参数、局部变量、返回地址等。 ` pcTaskName`：任务的名称，是一个字符串数组，用于标识任务的名称。

     ### 3.堆和栈的区别，在freertos中的作用

     在FreeRTOS中，堆（Heap）和栈（Stack）是两个不同的概念，用于存储不同类型的数据：

     1. 堆（Heap）：   

     2. 堆是通过动态内存分配来实现的一块内存区域，用于存储数据结构和对象。   

     3. 在FreeRTOS中，堆内存（Heap）由内存管理函数分配，如 `pvPortMalloc` 和 `vPortFree`。   - 堆内存用于存储任务控制块（Task Control Block，TCB）以及其他内核对象，如消息队列、信号量等。 

     4.  堆内存的大小可以通过配置参数来确定，根据系统需求和资源限制进行调整。

        

     5. 2. 栈（Stack）：   - 栈是用于存储函数调用的局部变量、函数参数和返回地址等临时数据的一块内存区域。   - 在FreeRTOS中，每个任务都有自己的栈空间，用于保存任务的执行上下文和局部变量。   - 任务栈的大小可以在任务创建时指定，根据任务的需求和硬件平台的限制进行配置。 总结来说，在FreeRTOS中，堆和栈是两个独立的内存区域，用于存储不同类型的数据。堆内存用于动态分配和管理内核对象，而任务栈则用于存储任务的执行上下文和局部变量。它们各自承担不同的功能和用途，并在任务管理和执行过程中起到重要的作用。

     ###  +

     

     

     ![image-20231201191617103](C:\Users\20215\AppData\Roaming\Typora\typora-user-images\image-20231201191617103.png)

```c

```

![image-20231201191733621](C:\Users\20215\AppData\Roaming\Typora\typora-user-images\image-20231201191733621.png)
