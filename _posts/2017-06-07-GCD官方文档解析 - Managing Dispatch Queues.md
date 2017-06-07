---
layout: post
title:  "GCD官方文档解析 - Managing Dispatch Queues"
date:  2017-06-07 17:46:00 +0800
categories: GCD iOS
---

上一篇文章简单介绍了下GCD，这里给大家讲解GCD中关于队列管理的一些知识，也是GCD最常用的部分。

# Managing Dispatch Queues

## Dispatch Queue Types

### 队列的类型

* 串行队列

  `DISPATCH_QUEUE_SERIAL`

  按先进先出的顺序处理传入的block。

* 并行队列

  `DISPATCH_QUEUE_CONCURRENT`

  传入的block被并发的处理，先传入的block不一定比后传入的block先完成，取决于之前block的处理时间，文档上还说虽然是并发处理的，但可以使用barrier block在并行队列中设置同步点，关于barrier block会在后面讲到。

### 创建队列

```objective-c
dispatch_queue_t dispatch_queue_create(const char *label, dispatch_queue_attr_t attr);
```

#### 参数：

* `const char *label`

  给队列一个标签，方便调试时区分队列，一般以反向DNS风格命名，如`"com.tc.queue"`，该参数可以为空（`NULL`）。

* `dispatch_queue_attr_t attr`

  这个属性指定队列类型`DISPATCH_QUEUE_SERIAL`为串行队列，`DISPATCH_QUEUE_CONCURRENT`为并行队列，该参数可为空（`NULL`），当该参数为空时，默认为串行队列，在iOS 4.3或更早版本上，只能指定该参数为空。

  * `DISPATCH_QUEUE_SERIAL`

    ```
    - (void)test
    {
        __block int index = 1;
        dispatch_queue_t queue_serial = dispatch_queue_create("com.tc.serial", DISPATCH_QUEUE_SERIAL);
        NSLog(@"开始运行");
        dispatch_async(queue_serial, ^{
            sleep(3);
            NSLog(@"第1个进入的Block, 第%i个出队列", index++);
        });
        dispatch_async(queue_serial, ^{
            NSLog(@"第2个进入的Block, 第%i个出队列", index++);
        });
        dispatch_async(queue_serial, ^{
            sleep(5);
            NSLog(@"第3个进入的Block, 第%i个出队列", index++);
        });
        dispatch_async(queue_serial, ^{
            sleep(1);
            NSLog(@"第4个进入的Block, 第%i个出队列", index++);
        });
    }
    ```

    输出（注意时间）：

    ```
    2017-04-28 18:18:47.732 demo[36802:2352075] 开始运行
    2017-04-28 18:18:50.736 demo[36802:2352122] 第1个进入的Block, 第1个出队列
    2017-04-28 18:18:50.737 demo[36802:2352122] 第2个进入的Block, 第2个出队列
    2017-04-28 18:18:55.740 demo[36802:2352122] 第3个进入的Block, 第3个出队列
    2017-04-28 18:18:56.744 demo[36802:2352122] 第4个进入的Block, 第4个出队列
    ```

  * `DISPATCH_QUEUE_CONCURRENT`

    ```
    - (void)test
    {
        __block int index = 1;
        dispatch_queue_t queue_concurrent = dispatch_queue_create("com.tc.concurrent", DISPATCH_QUEUE_CONCURRENT);
        NSLog(@"开始运行");
        dispatch_async(queue_concurrent, ^{
            sleep(3);
            NSLog(@"第1个进入的Block, 第%i个出队列", index++);
        });
        dispatch_async(queue_concurrent, ^{
            NSLog(@"第2个进入的Block, 第%i个出队列", index++);
        });
        dispatch_async(queue_concurrent, ^{
            sleep(5);
            NSLog(@"第3个进入的Block, 第%i个出队列", index++);
        });
        dispatch_async(queue_concurrent, ^{
            sleep(1);
            NSLog(@"第4个进入的Block, 第%i个出队列", index++);
        });
    }
    ```

    输出（注意时间）：

    ```
    2017-04-28 18:28:25.394 demo[36859:2359335] 开始运行
    2017-04-28 18:28:25.394 demo[36859:2359507] 第2个进入的Block, 第1个出队列
    2017-04-28 18:28:26.395 demo[36859:2359505] 第4个进入的Block, 第2个出队列
    2017-04-28 18:28:28.395 demo[36859:2359522] 第1个进入的Block, 第3个出队列
    2017-04-28 18:28:30.395 demo[36859:2359504] 第3个进入的Block, 第4个出队列
    ```


## Dispatch Queue Label Constants

### 获取队列Label

```
const char * dispatch_queue_get_label(dispatch_queue_t queue);
```

#### 参数：

* `dispatch_queue_t queue`

  需要获取Label的队列。

  如果需要获取当前队列的Label则使用`DISPATCH_CURRENT_QUEUE_LABEL`。

  ```
  - (void)test
  {
      dispatch_queue_t queue = dispatch_queue_create("com.tc.queue1", NULL);
      dispatch_sync(queue, ^{
          NSLog(@"%s", dispatch_queue_get_label(DISPATCH_CURRENT_QUEUE_LABEL));
      });
      NSLog(@"%s", dispatch_queue_get_label(DISPATCH_CURRENT_QUEUE_LABEL));
      NSLog(@"%s", dispatch_queue_get_label(queue));
  }
  ```

  输出：

  ```
  2017-05-02 11:16:57.771 demo[38100:2468155] com.tc.queue1
  2017-05-02 11:16:57.771 demo[38100:2468155] com.apple.main-thread
  2017-05-02 11:16:57.771 demo[38100:2468155] com.tc.queue1
  ```

## dispatch_queue_t

### 队列

```dui
typedef NSObject<OS_dispatch_queue> *dispatch_queue_t;
```

可见`dispatch_queue_t`其实是个对象，也很好的解释了在MRC下为何要手动管理`dispatch_queue_t`的内存。

Dispatch队列其实都是FIFO的，在串行队列上毋庸置疑。在并行队列上从表面上看并不符合FIFO的原则，但其实并行队列应该理解为n个并行的队列组合，组合中每个队列也是符合FIFO原则的，GCD帮助程序员管理了如何分配任务给这些组合里的队列。

## dispatch_get_main_queue

### 获取主队列

```
dispatch_queue_t dispatch_get_main_queue(void);
```

这个函数返回程序的主队列，这个队列在程序一开始时就被创建并与主线程关联，主队列是串行队列。

**注意：主线程与主队列并不等同**

主队列在在主线程被创建并关联的时机：

1. 调用`dispatch_main()`
2. 调用`UIApplicationMain`（iOS）或者`NSApplicationMain`（macOS）
3. 在主线程使用`CFRunLoopRef`

大多数情况下我们的App会在`mian()`函数里使用第2种方式。

对主队列使用`dispatch_suspend`、`dispatch_resume`、`dispatch_set_context`是无效的。

## dispatch_get_global_queue

### 获取全局并行队列

返回系统预定义的全局并行队列。

```
dispatch_queue_t dispatch_get_global_queue(long identifier, unsigned long flags);
```

#### 参数：

* `long identifier`

  全局队列的ID，区别就在于优先级。

  iOS8.0或以上版本：

  * QOS_CLASS_USER_INTERACTIVE
  * QOS_CLASS_USER_INITIATED
  * QOS_CLASS_DEFAULT
  * QOS_CLASS_UTILITY
  * QOS_CLASS_BACKGROUND
  * QOS_CLASS_UNSPECIFIED

  iOS8以下的版本：

  * DISPATCH_QUEUE_PRIORITY_HIGH
  * DISPATCH_QUEUE_PRIORITY_DEFAULT
  * DISPATCH_QUEUE_PRIORITY_LOW
  * DISPATCH_QUEUE_PRIORITY_BACKGROUND

  tips：在background队列中，对I/O操作进行了优化，具体可参考<a href="http://www.cocoachina.com/ios/20170105/18525.html" target="_blank">iOS编程中throttle那些事</a>。

* `unsigned long flags`

  为未来的API预留的参数，现在不用管，直接传0。

对系统预定义队列使用`dispatch_suspend`、`dispatch_resume`、`dispatch_set_context`是无效的。

## dispatch_set_target_queue

### 给目标对象设置一个队列

```
void dispatch_set_target_queue(dispatch_object_t object, dispatch_queue_t queue);
```

将一个dispatch对象设置一个队列

#### 参数：

* `object`

  将要设置的对象，不能为空。

* `queue`

  `object`的新目标队列，设置了目标权限后`queue`的引用计数+1会被保留，同时之前设置的`queue`会被release。

这个函数有以下几种用途：

* 将queue作为object参数传入

  * 设置queue的优先级

    * demo 1

      ```
      - (void)test
      {
          dispatch_queue_t tc_queue = dispatch_queue_create("com.tc.queue", DISPATCH_QUEUE_SERIAL);
          dispatch_queue_t default_queue = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
          dispatch_queue_t user_interactive_queue = dispatch_get_global_queue(QOS_CLASS_USER_INTERACTIVE, 0);
          dispatch_set_target_queue(tc_queue, user_interactive_queue);
          dispatch_async(tc_queue, ^{
              double x = MAXFLOAT;
              for(long i = 0; i < 999999999; i++)
              {
                  x = x / 1.454;
              }
              NSLog(@"tc_queue");
          });
          dispatch_async(default_queue, ^{
              double x = MAXFLOAT;
              for(long i = 0; i < 999999999; i++)
              {
                  x = x / 1.454;
              }
              NSLog(@"default_queue");
          });
          NSLog(@"start");
      }
      ```

      输出（注意时间）：

      ```
      2017-06-07 16:48:52.391 demo[2494:44948437] start
      2017-06-07 16:49:49.425 demo[2494:44948641] tc_queue
      2017-06-07 16:49:50.355 demo[2494:44948642] default_queue
      ```

    * demo 2

      ```
      - (void)test
      {
          dispatch_queue_t tc_queue = dispatch_queue_create("com.tc.queue", DISPATCH_QUEUE_SERIAL);
          dispatch_queue_t default_queue = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
          dispatch_queue_t user_interactive_queue = dispatch_get_global_queue(QOS_CLASS_USER_INTERACTIVE, 0);
          dispatch_set_target_queue(tc_queue, default_queue);
          dispatch_async(tc_queue, ^{
              double x = MAXFLOAT;
              for(long i = 0; i < 999999999; i++)
              {
                  x = x / 1.454;
              }
              NSLog(@"tc_queue");
          });
          dispatch_async(user_interactive_queue, ^{
              double x = MAXFLOAT;
              for(long i = 0; i < 999999999; i++)
              {
                  x = x / 1.454;
              }
              NSLog(@"user_interactive_queue");
          });
          NSLog(@"start");
      }
      ```

      输出（注意时间）：

      ```
      2017-06-07 16:52:49.254 demo[2523:45002465] start
      2017-06-07 16:53:45.850 demo[2523:45002655] user_interactive_queue
      2017-06-07 16:53:46.811 demo[2523:45002661] tc_queue
      ```

  * 将若干串行队列组装成一个串行队列

    一般情况下，向几个串行队列分别提交block，block的执行其实在这几个串行队列中是并行的，没有先后顺序，使用`dispatch_set_target_queue`后，可以将串行队列串联起来，可以想象成原来三根吸管吸果汁时，三根吸管同时有果汁到嘴里，现在我们把三根吸管连接起来，嗯大概就是这么个意思，再多想象一点，三根吸管上有很多小洞，离嘴最近习惯上的小洞会更先把果汁送到嘴里。

    ```
    - (void)test
    {
        dispatch_queue_t tc_queue = dispatch_queue_create("com.tc.queue", DISPATCH_QUEUE_SERIAL);
        dispatch_queue_t queue_1 = dispatch_queue_create("com.tc.queue.1", DISPATCH_QUEUE_SERIAL);
        dispatch_queue_t queue_2 = dispatch_queue_create("com.tc.queue.2", DISPATCH_QUEUE_SERIAL);
        dispatch_queue_t queue_3 = dispatch_queue_create("com.tc.queue.3", DISPATCH_QUEUE_SERIAL);
        dispatch_set_target_queue(queue_1, tc_queue);
        dispatch_set_target_queue(queue_2, tc_queue);
        dispatch_set_target_queue(queue_3, tc_queue);
        dispatch_async(queue_1, ^{
            
        });
        dispatch_async(queue_3, ^{
            
        });
        dispatch_async(queue_3, ^{
            NSLog(@"1");
        });
        dispatch_async(queue_2, ^{
            NSLog(@"2");
        });
        dispatch_async(queue_1, ^{
            NSLog(@"3");
        });
        dispatch_async(queue_3, ^{
            NSLog(@"4");
        });
        dispatch_async(queue_2, ^{
            NSLog(@"5");
        });
        dispatch_async(queue_1, ^{
            NSLog(@"6");
        });
    }
    ```

    输出：

    ```
    2017-06-07 17:02:51.065 demo[2631:45141955] 3
    2017-06-07 17:02:51.066 demo[2631:45141955] 6
    2017-06-07 17:02:51.066 demo[2631:45141955] 1
    2017-06-07 17:02:51.066 demo[2631:45141955] 4
    2017-06-07 17:02:51.066 demo[2631:45141955] 2
    2017-06-07 17:02:51.066 demo[2631:45141955] 5
    ```

    需要注意的是，如果使用这种方法，必须注意在同一个target的队列中不要互相同步调用，会死锁。

* 将source作为object参数传入

  更换source对应的queue。

  ```

  ```

  ​

  ​