## 线程安全

#### 

说到多线程就不得不提多线程中的锁机制，多线程操作过程中往往多个线程是并发执行的，同一个资源可能被多个线程同时访问，造成资源抢夺，这个过程中如果没有锁机制往往会造成重大问题。举例来说，每年春节都是一票难求，在12306买票的过程中，成百上千的票瞬间就消失了。不妨假设某辆车有1千张票，同时有几万人在抢这列车的车票，顺利的话前面的人都能买到票。但是如果现在只剩下一张票了，而同时还有几千人在购买这张票，虽然在进入购票环节的时候会判断当前票数，但是当前已经有100个线程进入购票的环节，每个线程处理完票数都会减1,100个线程执行完当前票数为-99，遇到这种情况很明显是不允许的。

要解决资源抢夺问题在iOS中有常用的有两种方法：一种是使用NSLock同步锁，另一种是使用@synchronized代码块。两种方法实现原理是类似的，只是在处理上代码块使用起来更加简单（C#中也有类似的处理机制synchronized和lock）。

## 多线程的安全隐患
#### 资源共享
一块资源可能会被多个线程共享，也就是多个线程可能会访问同一块资源。比如多个线程访问同一个对象、同一个变量、同一个文件。当多个线程访问同一块资源时，很容易引发数据错乱和数据安全问题。

​      

```objective-c
- (void)test {
    //默认有20张票
    leftTicketsCount = 10;

    //开启多个线程，模拟售票员售票
    NSThread *thread1=[[NSThread alloc]initWithTarget:self selector:@selector(sellTickets) object:nil];
    thread1.name=@"售票员A";

    NSThread *thread2=[[NSThread alloc]initWithTarget:self selector:@selector(sellTickets) object:nil];
    thread2.name=@"售票员B";

    NSThread *thread3=[[NSThread alloc]initWithTarget:self selector:@selector(sellTickets) object:nil];
    thread3.name=@"售票员C";

    //开启线程

    [thread1 start];
    [thread2 start];
    [thread3 start];
}

- (void)sellTickets {
    while (1) {
        //1.先检查票数
        int count = leftTicketsCount;
        if (count>0) {
            //暂停一段时间
            [NSThread sleepForTimeInterval:0.002];
      //2.票数-1
      leftTicketsCount= count-1;
      
      //获取当前线程
      NSThread *current=[NSThread currentThread];
      NSLog(@"%@--卖了一张票，还剩余%d张票", current.name, leftTicketsCount);
  }
  else {
      //退出线程
      [NSThread exit];
  }
 }
}
```



打印结果：

![售票结果，出现了重复买票的现象](image\10.png)

## 如何解决

#### 1.使用互斥锁
格式：
>@synchronized(锁对象) { 
// 需要锁定的代码  
}

注意：锁定1份代码只用1把锁，用多把锁是无效的

```objective-c

- (void)sellTickets
{
    while (1) {
        @synchronized(self){//加一把锁
            //1.先检查票数
            int count = leftTicketsCount;
            if (count>0) {
                //暂停一段时间
                [NSThread sleepForTimeInterval:0.002];
            //2.票数-1
            leftTicketsCount= count-1;
            
            //获取当前线程
            NSThread *current=[NSThread currentThread];
            NSLog(@"%@--卖了一张票，还剩余%d张票", current.name, leftTicketsCount);
        }
        else {
            //退出线程
            [NSThread exit];
        }
    }
}

```

打印结果：

![Paste_Image.png](image\11.png)

互斥锁的优缺点
优点：能有效防止因多线程抢夺资源造成的数据安全问题
缺点：需要消耗大量的CPU资源

互斥锁的使用前提：多条线程抢夺同一块资源 
相关专业术语：线程同步,多条线程按顺序地执行任务
互斥锁，就是使用了线程同步技术



#### 2.原子和非原子属性

OC在定义属性时有nonatomic和atomic两种选择
atomic：原子属性，为setter方法加锁（默认就是atomic）
nonatomic：非原子属性，不会为setter方法加锁

atomic加锁原理：

```objective-c
@property (assign, atomic) int age;

- (void)setAge:(int)age
{ 
    @synchronized(self) { 
       _age = age;
    }
}
```


nonatomic和atomic对比
* atomic：线程安全，需要消耗大量的资源
* nonatomic：非线程安全，适合内存小的移动设备

#### 3.NSLock
![NSLock使用效果](image\12.png)

#### 4.dispatch_semaphore

![dispatch_semaphore使用效果](image\13.png)
> [# 深入理解 iOS 开发中的锁](https://twitter.com/intent/tweet?text=%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3%20iOS%20%E5%BC%80%E5%8F%91%E4%B8%AD%E7%9A%84%E9%94%81%20%C2%BB&hashtags=&url=https://bestswifter.com/ios-lock/ "Tweet '深入理解 iOS 开发中的锁'")

