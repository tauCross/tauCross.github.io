---
layout: post
title:  "GCD官方文档解析 - Managing Dispatch Queues"
date:  2017-06-07 17:46:00 +0800
categories: GCD iOS
---

上一篇文章简单介绍了下GCD，这里给大家讲解GCD中关于队列管理的一些知识，也是GCD最常用的部分。

# Managing Dispatch Queues

## Dispatch Queue Types

队列的类型

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

获取队列Label

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

队列

```dui
typedef NSObject<OS_dispatch_queue> *dispatch_queue_t;
```

可见`dispatch_queue_t`其实是个对象，也很好的解释了在MRC下为何要手动管理`dispatch_queue_t`的内存。

Dispatch队列其实都是FIFO的，在串行队列上毋庸置疑。在并行队列上从表面上看并不符合FIFO的原则，但其实并行队列应该理解为n个并行的队列组合，组合中每个队列也是符合FIFO原则的，GCD帮助程序员管理了如何分配任务给这些组合里的队列。

## dispatch_get_main_queue

获取主队列

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

给目标对象设置一个队列

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
  - (void)test
  {
      dispatch_queue_t tc_queue_1 = dispatch_queue_create("com.tc.queue.1", DISPATCH_QUEUE_SERIAL);
      dispatch_queue_t tc_queue_2 = dispatch_queue_create("com.tc.queue.2", DISPATCH_QUEUE_SERIAL);
      dispatch_source_t tc_source = dispatch_source_create(DISPATCH_SOURCE_TYPE_DATA_ADD, 0, 0, tc_queue_1);
      dispatch_source_set_event_handler(tc_source, ^{
          NSLog(@"%li, %s", dispatch_source_get_data(tc_source), dispatch_queue_get_label(DISPATCH_CURRENT_QUEUE_LABEL));
      });
      dispatch_resume(tc_source);
      dispatch_source_merge_data(tc_source, 1);
      dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
          dispatch_set_target_queue(tc_source, tc_queue_2);
          dispatch_source_merge_data(tc_source, 2);
      });
  }
  ```

  输出：

  ```
  2017-06-08 12:36:50.176 demo[4440:46508136] 1, com.tc.queue.1
  2017-06-08 12:36:51.176 demo[4440:46508137] 2, com.tc.queue.2
  ```

* 将io作为object参数传入

  根据文档上的描述，把io作为object参数传入能够改变io操作的优先级，比如目标队列的优先级是`DISPATCH_QUEUE_PRIORITY_BACKGROUND`的时候，io操作在系统资源紧张的时候会受到节流器的限制。

  对于这块的内容将放在后面关于`dispatch_io_t`的内容中详细描述。

## dispatch_async

提交一个异步执行的block到队列里，立即执行。这个函数会立刻返回（非阻塞），所以不影响后面代码的运行。

由于队列的特性，先提交的block会更早开始执行（开始-执行中-结束），在并行队列中更早提交的block不一定更早结束，而在单个的串行队列中，更早提交的block只有在执行完毕后才会执行下一个block。

```
void dispatch_async(dispatch_queue_t queue, dispatch_block_t block);
```

#### 参数：

* `queue`

  block想要提交的目标队列。这个队列会被系统retain直到block运行完成，参数不可为空。

* `block`

  block会被自动copy与release。

```
- (void)test
{
    dispatch_queue_t tc_queue = dispatch_queue_create("com.tc.queue", DISPATCH_QUEUE_SERIAL);
    NSLog(@"0");
    dispatch_async(tc_queue, ^{
        NSLog(@"1");
    });
    dispatch_async(tc_queue, ^{
        NSLog(@"2");
    });
    NSLog(@"3");
}
```

输出：

```
2017-06-08 14:57:01.658 demo[5065:47129970] 0
2017-06-08 14:57:01.659 demo[5065:47129970] 3
2017-06-08 14:57:01.659 demo[5065:47130176] 1
2017-06-08 14:57:01.659 demo[5065:47130176] 2
```

## dispatch_async_f

一脸懵逼，看着挺吓人的，其实就是将一个C函数当做一个block丢进去异步执行，与`dispatch_async`差不多。

```
void dispatch_async_f(dispatch_queue_t queue, void *context, dispatch_function_t work);
```

#### 参数：

* `queue`

  想要提交的目标队列。

* `context`

  后面那个work的参数。

* `wrok`

  work是`dispatch_function_t`类型的，来看下`dispatch_function_t`到底是啥？

  ```
  typedef void (*dispatch_function_t)(void *_Nullable);
  ```

  `dispatch_function_t`就是个`void(*)()`类型的函数指针。

```
- (void)test
{
    dispatch_queue_t tc_queue = dispatch_queue_create("com.tc.queue", DISPATCH_QUEUE_SERIAL);
    NSLog(@"0");
    dispatch_async_f(tc_queue, NULL, tc);
    NSLog(@"2");
}

void tc()
{
    NSLog(@"1");
}
```

输出：

```
2017-06-08 16:04:48.242 demo[5490:48047689] 0
2017-06-08 16:04:48.243 demo[5490:48047689] 2
2017-06-08 16:04:48.243 demo[5490:48047881] 1
```

## dispatch_sync

写到这里的时候我对自己是抱有怀疑态度的，这个函数官方文档居然特么的没写！！？难道我之前用的都是私有API？瑟瑟发抖。

这个对应于`dispatch_async`异步执行，`dispatch_sync`是同步执行，这个函数不会立刻返回，而是等提交的block都执行完毕后再返回（阻塞），因为这个特性，使用`dispatch_sync`时得注意死锁的问题。

```
dispatch_sync(dispatch_queue_t queue, DISPATCH_NOESCAPE dispatch_block_t block);
```

#### 参数：

* `queue`

  同`dispatch_aync`

* `block`

  同`dispatch_aync`

```
- (void)test
{
    dispatch_queue_t tc_queue = dispatch_queue_create("com.tc.queue", DISPATCH_QUEUE_SERIAL);
    NSLog(@"0");
    dispatch_sync(tc_queue, ^{
        NSLog(@"1");
    });
    dispatch_sync(tc_queue, ^{
        NSLog(@"2");
    });
    NSLog(@"3");
}
```

输出：

```
2017-06-08 15:59:34.024 demo[5458:47975829] 0
2017-06-08 15:59:34.025 demo[5458:47975829] 1
2017-06-08 15:59:34.025 demo[5458:47975829] 2
2017-06-08 15:59:34.025 demo[5458:47975829] 3
```

## dispatch_sync_f

道理跟`dispatch_asyn_f`一样，不再赘述。

## dispatch_after

也是常用的函数之一，用于在指定时间将block进行入队操作，所以传入的block不会被立刻执行，而是在指定时间在会被加入队列。可以指定当前时间更早的时间来实现一直等待进入队列，但这不是最佳实践，可以用`DISPATCH_TIME_FOREVER`来替代。

```
void dispatch_after(dispatch_time_t when, dispatch_queue_t queue, dispatch_block_t block);
```

#### 参数：

* `when`

  一个由`dispatch_time`函数或`dispatch_walltime`函数返回的`dispatch_time_t`，说人话就是指定block延时多久执行。

* `queue`

  同`dispatch_async`

* `block`

  同`dispatch_aync`

```
- (void)test
{
    dispatch_queue_t tc_queue = dispatch_queue_create("com.tc.queue", DISPATCH_QUEUE_SERIAL);
    dispatch_async(tc_queue, ^{
        NSLog(@"0");
    });
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1 * NSEC_PER_SEC)), tc_queue, ^{
        NSLog(@"1");
    });
    dispatch_async(tc_queue, ^{
        NSLog(@"2");
    });
}
```

输出：

```
2017-06-08 16:37:15.405 demo[5730:48486382] 0
2017-06-08 16:37:15.406 demo[5730:48486382] 2
2017-06-08 16:37:16.405 demo[5730:48486383] 1
```

## dispatch_after_f

道理跟`dispatch_asyn_f`一样，不再赘述。

## dispatch_apply

再次懵逼，这个函数官方文档里也没有！？！

将一个block加入一个队列多次，并等待这些block全部完成再返回，所以这个函数是同步的（阻塞）。

如果指定的队列是并发队列，则这些block的调用可能也是并发执行的，所以这里需要关注可重入性。

可重入性相关的文章推荐<a href="http://www.jianshu.com/p/8a88fbe3e1a9" target="_blank">可重入与线程安全</a>

```
dispatch_apply(size_t iterations, dispatch_queue_t queue, DISPATCH_NOESCAPE void (^block)(size_t));
```

#### 参数：

* `iterations`

  block加入队列的次数

* `queue`

  同`dispatch_aync`

* `block`

  有个类型为`size_t`的返回参数，用以告诉block这是第几个调用。

```
- (void)test
{
    dispatch_queue_t tc_queue_concurrent = dispatch_queue_create("com.tc.queue.concurrent", DISPATCH_QUEUE_CONCURRENT);
    dispatch_queue_t tc_queue_serial = dispatch_queue_create("com.tc.queue.serial", DISPATCH_QUEUE_SERIAL);
    NSLog(@"concurrent");
    dispatch_apply(10, tc_queue_concurrent, ^(size_t index) {
        NSLog(@"%lu", index);
    });
    NSLog(@"serial");
    dispatch_apply(10, tc_queue_serial, ^(size_t index) {
        NSLog(@"%lu", index);
    });
}
```

输出：

```
2017-06-08 17:07:31.582 demo[5912:48898006] concurrent
2017-06-08 17:07:31.582 demo[5912:48898185] 0
2017-06-08 17:07:31.582 demo[5912:48898186] 2
2017-06-08 17:07:31.582 demo[5912:48898192] 1
2017-06-08 17:07:31.582 demo[5912:48898006] 3
2017-06-08 17:07:31.582 demo[5912:48898229] 4
2017-06-08 17:07:31.583 demo[5912:48898232] 5
2017-06-08 17:07:31.583 demo[5912:48898230] 6
2017-06-08 17:07:31.583 demo[5912:48898231] 7
2017-06-08 17:07:31.583 demo[5912:48898185] 8
2017-06-08 17:07:31.583 demo[5912:48898186] 9
2017-06-08 17:07:31.583 demo[5912:48898006] serial
2017-06-08 17:07:31.583 demo[5912:48898006] 0
2017-06-08 17:07:31.583 demo[5912:48898006] 1
2017-06-08 17:07:31.583 demo[5912:48898006] 2
2017-06-08 17:07:31.584 demo[5912:48898006] 3
2017-06-08 17:07:31.584 demo[5912:48898006] 4
2017-06-08 17:07:31.584 demo[5912:48898006] 5
2017-06-08 17:07:31.584 demo[5912:48898006] 6
2017-06-08 17:07:31.584 demo[5912:48898006] 7
2017-06-08 17:07:31.584 demo[5912:48898006] 8
2017-06-08 17:07:31.584 demo[5912:48898006] 9
```

## dispatch_apply_f

道理跟`dispatch_asyn_f`一样，不再赘述。

## dispatch_queue_get_label

获取队列在创建时指定的label。

```
const char * dispatch_queue_get_label(dispatch_queue_t queue);
```

#### 参数：

* `queue`

  需要获取label的queue，如果需要获取当前queue的label则直接传`DISPATCH_CURRENT_QUEUE_LABEL`

```
- (void)test
{
    dispatch_queue_t tc_queue_concurrent = dispatch_queue_create("com.tc.queue.concurrent", DISPATCH_QUEUE_CONCURRENT);
    dispatch_queue_t tc_queue_serial = dispatch_queue_create("com.tc.queue.serial", DISPATCH_QUEUE_SERIAL);
    dispatch_async(tc_queue_concurrent, ^{
        NSLog(@"%s", dispatch_queue_get_label(tc_queue_serial));
        NSLog(@"%s", dispatch_queue_get_label(DISPATCH_CURRENT_QUEUE_LABEL));
    });
}
```

输出：

```
2017-06-08 17:12:54.167 demo[5940:48971595] com.tc.queue.serial
2017-06-08 17:12:54.167 demo[5940:48971595] com.tc.queue.concurrent
```

## dispatch_get_specific

获取当前队列的标识。最有用的就是防止锁死了，关于`dispatch_get_current_queue`被弃用与之后如何防止锁死推荐念茜的博客<a href="http://blog.csdn.net/yiyaaixuexi/article/details/17752925" target="_blank">被废弃的dispatch_get_current_queue</a>

```
void * dispatch_get_specific(const void *key);
```

#### 参数：

* `key`

  标识的key。

demo留在`dispatch_queue_set_specific`

## dispatch_queue_set_specific

设置队列的标识。

```
void dispatch_queue_set_specific(dispatch_queue_t queue, const void *key, void *context, dispatch_function_t destructor);
```

#### 参数：

* `queue`

  想要加标识的队列。

* `key`

  标识的key。

* `context`

  标识关联的上下文，可为空。

* `destructor`

  上下文的析构函数，可为空，当上下文为空时，这个析构函数会被忽略。

```
- (void)test
{
    static char key;
    CFStringRef context = CFSTR("123");
    dispatch_queue_t tc_queue = dispatch_queue_create("com.tc.queue", DISPATCH_QUEUE_SERIAL);
    dispatch_queue_set_specific(tc_queue, &key, (void *)context, tc_destructor);
    dispatch_async(tc_queue, ^{
        if(dispatch_get_specific(&key))
        {
            NSLog(@"1");
        }
    });
}

void tc_destructor(void *context)
{
    CFRelease(context);
    NSLog(@"2");
}
```

输出：

```
2017-06-08 17:48:55.721 demo[6182:49461837] 1
2017-06-08 17:48:55.722 demo[6182:49461835] 2
```

