## 概览
大家都知道，在开发过程中应该尽可能减少用户等待时间，让程序尽可能快的完成运算。可是无论是哪种语言开发的程序最终往往转换成汇编语言进而解释成机器码来执行。但是机器码是按顺序执行的，一个复杂的多步操作只能一步步按顺序逐个执行。改变这种状况可以从两个角度出发：对于单核处理器，可以将多个步骤放到不同的线程，这样一来用户完成UI操作后其他后续任务在其他线程中，当CPU空闲时会继续执行，而此时对于用户而言可以继续进行其他操作；对于多核处理器，如果用户在UI线程中完成某个操作之后，其他后续操作在别的线程中继续执行，用户同样可以继续进行其他UI操作，与此同时前一个操作的后续任务可以分散到多个空闲CPU中继续执行（当然具体调度顺序要根据程序设计而定），及解决了线程阻塞又提高了运行效率。苹果从iPad2 开始使用双核A5处理器（iPhone中从iPhone 4S开始使用），A7中还加入了协处理器，如何充分发挥这些处理器的性能确实值得思考。

## 简介
当用户播放音频、下载资源、进行图像处理时往往希望做这些事情的时候其他操作不会被中断或者希望这些操作过程中更加顺畅。在单线程中一个线程只能做一件事情，一件事情处理不完另一件事就不能开始，这样势必影响用户体验。早在单核处理器时期就有多线程，这个时候多线程更多的用于解决线程阻塞造成的用户等待（通常是操作完UI后用户不再干涉，其他线程在等待队列中，CPU一旦空闲就继续执行，不影响用户其他UI操作），其处理能力并没有明显的变化。如今无论是移动操作系统还是PC、服务器都是多核处理器，于是“并行运算”就更多的被提及。一件事情我们可以分成多个步骤，在没有顺序要求的情况下使用多线程既能解决线程阻塞又能充分利用多核处理器运行能力。

下图反映了一个包含8个操作的任务在一个有两核心的CPU中创建四个线程运行的情况。假设每个核心有两个线程，那么每个CPU中两个线程会交替执行，两个CPU之间的操作会并行运算。单就一个CPU而言两个线程可以解决线程阻塞造成的不流畅问题，其本身运行效率并没有提高，多CPU的并行运算才真正解决了运行效率问题，这也正是并发和并行的区别。当然，不管是多核还是单核开发人员不用过多的担心，因为任务具体分配给几个CPU运算是由系统调度的，开发人员不用过多关心系统有几个CPU。开发人员需要关心的是线程之间的依赖关系，因为有些操作必须在某个操作完成完才能执行，如果不能保证这个顺序势必会造成程序问题。

![多线程](image\6.png)

## iOS多线程

在iOS中每个进程启动后都会建立一个主线程（UI线程），这个线程是其他线程的父线程。由于在iOS中除了主线程，其他子线程是独立于Cocoa Touch的，所以只有主线程可以更新UI界面（新版iOS中，使用其他线程更新UI可能也能成功，但是不推荐）。iOS中多线程使用并不复杂，关键是如何控制好各个线程的执行顺序、处理好资源竞争问题。常用的多线程开发有三种方式：

* 1.NSThread
* 2.NSOperation
* 3.GCD

三种方式是随着iOS的发展逐渐引入的，所以相比而言后者比前者更加简单易用，并且GCD也是目前苹果官方比较推荐的方式（它充分利用了多核处理器的运算性能）。

#### NSThread

NSThread是轻量级的多线程开发，使用起来也并不复杂，但是使用NSThread需要自己管理线程生命周期。可以使用对象方法
> +(void)detachNewThreadSelector:(SEL)selector toTarget:(id)target withObject:(id)argument

直接将操作添加到线程中并启动，也可以使用对象方法
> -(instancetype)initWithTarget:(id)target selector:(SEL)selector object:(id)argument

创建一个线程对象，然后调用start方法启动线程。

###### 解决线程阻塞问题

在资源下载过程中，由于网络原因有时候很难保证下载时间，如果不使用多线程可能用户完成一个下载操作需要长时间的等待，这个过程中无法进行其他操作。下面演示一个采用多线程下载图片的过程，在这个示例中点击按钮会启动一个线程去下载图片，下载完成后使用UIImageView将图片显示到界面中。可以看到用户点击完下载按钮后，不管图片是否下载完成都可以继续操作界面，不会造成阻塞。

```objective-c
//
//  NSThread实现多线程
//  MultiThread
//
//  Created by Kenshin Cui on 14-3-22.
//  Copyright (c) 2014年 Kenshin Cui. All rights reserved.
//

#import "KCMainViewController.h"

@interface KCMainViewController (){
    UIImageView *_imageView;
}

@end

@implementation KCMainViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self layoutUI];
}

#pragma mark 界面布局
-(void)layoutUI{
    _imageView =[[UIImageView alloc]initWithFrame:[UIScreen mainScreen].applicationFrame];
    _imageView.contentMode=UIViewContentModeScaleAspectFit;
    [self.view addSubview:_imageView];
    
    UIButton *button=[UIButton buttonWithType:UIButtonTypeRoundedRect];
    button.frame=CGRectMake(50, 500, 220, 25);
    [button setTitle:@"加载图片" forState:UIControlStateNormal];
    //添加方法
    [button addTarget:self action:@selector(loadImageWithMultiThread) forControlEvents:UIControlEventTouchUpInside];
    [self.view addSubview:button];
}

#pragma mark 将图片显示到界面
-(void)updateImage:(NSData *)imageData{
    UIImage *image=[UIImage imageWithData:imageData];
    _imageView.image=image;
}

#pragma mark 请求图片数据
-(NSData *)requestData{
    //对于多线程操作建议把线程操作放到@autoreleasepool中
    @autoreleasepool {
        NSURL *url=[NSURL URLWithString:@"http://images.apple.com/iphone-6/overview/images/biggest_right_large.png"];
        NSData *data=[NSData dataWithContentsOfURL:url];
        return data;
    }
}

#pragma mark 加载图片
-(void)loadImage{
    //请求数据
    NSData *data= [self requestData];
    /*将数据显示到UI控件,注意只能在主线程中更新UI,
     另外performSelectorOnMainThread方法是NSObject的分类方法，每个NSObject对象都有此方法，
     它调用的selector方法是当前调用控件的方法，例如使用UIImageView调用的时候selector就是UIImageView的方法
     Object：代表调用方法的参数,不过只能传递一个参数(如果有多个参数请使用对象进行封装)
     waitUntilDone:是否线程任务完成执行
    */
    [self performSelectorOnMainThread:@selector(updateImage:) withObject:data waitUntilDone:YES];
}

#pragma mark 多线程下载图片
-(void)loadImageWithMultiThread{
    //方法1：使用对象方法
    //创建一个线程，第一个参数是请求的操作，第二个参数是操作方法的参数
//    NSThread *thread=[[NSThread alloc]initWithTarget:self selector:@selector(loadImage) object:nil];
//    //启动一个线程，注意启动一个线程并非就一定立即执行，而是处于就绪状态，当系统调度时才真正执行
//    [thread start];
    
    //方法2：使用类方法
    [NSThread detachNewThreadSelector:@selector(loadImage) toTarget:self withObject:nil];
}
@end
```

运行效果：

![NSThreadEffect](image\7.png)

程序比较简单，但是需要注意执行步骤：当点击了“加载图片”按钮后启动一个新的线程，这个线程在演示中大概用了5s左右，在这5s内UI线程是不会阻塞的，用户可以进行其他操作，大约5s之后图片下载完成，此时调用UI线程将图片显示到界面中（这个过程瞬间完成）。另外前面也提到过，更新UI的时候使用UI线程，这里调用了NSObject的分类扩展方法，调用UI线程完成更新。


##### 扩展--NSObject分类扩展方法

为了简化多线程开发过程，苹果官方对NSObject进行分类扩展(本质还是创建NSThread)，对于简单的多线程操作可以直接使用这些扩展方法。
> -(void)performSelectorInBackground:(SEL)aSelector withObject:(id)arg：

在后台执行一个操作，本质就是重新创建一个线程执行当前方法。

> -(void)performSelector:(SEL)aSelector onThread:(NSThread *)thr withObject:(id)arg waitUntilDone:(BOOL)wait

在指定的线程上执行一个方法，需要用户创建一个线程对象。

> -(void)performSelectorOnMainThread:(SEL)aSelector withObject:(id)arg waitUntilDone:(BOOL)wait

在主线程上执行一个方法。


## NSOperation
使用NSOperation和NSOperationQueue进行多线程开发类似于C#中的线程池，只要将一个NSOperation（实际开中需要使用其子类NSInvocationOperation、NSBlockOperation）放到NSOperationQueue这个队列中线程就会依次启动。NSOperationQueue负责管理、执行所有的NSOperation，在这个过程中可以更加容易的管理线程总数和控制线程之间的依赖关系。

NSOperation有两个常用子类用于创建线程操作：NSInvocationOperation和NSBlockOperation，两种方式本质没有区别，但是是后者使用Block形式进行代码组织，使用相对方便。

#### NSInvocationOperation

首先使用NSInvocationOperation进行一张图片的加载演示，整个过程就是：创建一个操作，在这个操作中指定调用方法和参数，然后加入到操作队列。其他代码基本不用修改，直接修加载图片方法如下：
```objective-c
-(void)loadImageWithMultiThread{
    /*创建一个调用操作
     object:调用方法参数
    */
    NSInvocationOperation *invocationOperation=[[NSInvocationOperation alloc]initWithTarget:self selector:@selector(loadImage) object:nil];
    //创建完NSInvocationOperation对象并不会调用，它由一个start方法启动操作，但是注意如果直接调用start方法，则此操作会在主线程中调用，一般不会这么操作,而是添加到NSOperationQueue中
//    [invocationOperation start];
    
    //创建操作队列
    NSOperationQueue *operationQueue=[[NSOperationQueue alloc]init];
    //注意添加到操作队后，队列会开启一个线程执行此操作
    [operationQueue addOperation:invocationOperation];
}
```

#### NSBlockOperation
下面采用NSBlockOperation创建多个线程加载图片。
```objective-c
#pragma mark 多线程下载图片
-(void)loadImageWithMultiThread{
    int count=ROW_COUNT*COLUMN_COUNT;
    //创建操作队列
    NSOperationQueue *operationQueue=[[NSOperationQueue alloc]init];
    operationQueue.maxConcurrentOperationCount=5;//设置最大并发线程数
    //创建多个线程用于填充图片
    for (int i=0; i&lt;count; ++i){
        //方法1：创建操作块添加到队列
//        //创建多线程操作
//        NSBlockOperation *blockOperation=[NSBlockOperation blockOperationWithBlock:^{
//            [self loadImage:[NSNumber numberWithInt:i]];
//        }];
//        //创建操作队列
//
//        [operationQueue addOperation:blockOperation];
        
        //方法2：直接使用操队列添加操作
        [operationQueue addOperationWithBlock:^{
            [self loadImage:[NSNumber numberWithInt:i]];
        }];
    }
}
```

对比之前NSThread加载张图片很发现核心代码简化了不少，这里着重强调两点：

* 使用NSBlockOperation方法，所有的操作不必单独定义方法，同时解决了只能传递一个参数的问题。
* 调用主线程队列的addOperationWithBlock:方法进行UI更新，不用再定义一个参数实体（之前必须定义一个KCImageData解决只能传递一个参数的问题）。
* 使用NSOperation进行多线程开发可以设置最大并发线程，有效的对线程进行了控制（上面的代码运行起来你会发现打印当前进程时只有有限的线程被创建，如上面的代码设置最大线程数为5，则图片基本上是五个一次加载的）。

#### 线程执行顺序

前面使用NSThread很难控制线程的执行顺序，但是使用NSOperation就容易多了，每个NSOperation可以设置依赖线程。假设操作A依赖于操作B，线程操作队列在启动线程时就会首先执行B操作，然后执行A。对于前面优先加载最后一张图的需求，只要设置前面的线程操作的依赖线程为最后一个操作即可。修改图片加载方法如下：

```objective-c
-(void)loadImageWithMultiThread{
    int count=ROW_COUNT*COLUMN_COUNT;
    //创建操作队列
    NSOperationQueue *operationQueue=[[NSOperationQueue alloc]init];
    operationQueue.maxConcurrentOperationCount=5;//设置最大并发线程数
    
    NSBlockOperation *lastBlockOperation=[NSBlockOperation blockOperationWithBlock:^{
        [self loadImage:[NSNumber numberWithInt:(count-1)]];
    }];
    //创建多个线程用于填充图片
    for (int i=0; i&lt;count-1; ++i) {
        //方法1：创建操作块添加到队列
        //创建多线程操作
        NSBlockOperation *blockOperation=[NSBlockOperation blockOperationWithBlock:^{
            [self loadImage:[NSNumber numberWithInt:i]];
        }];
        //设置依赖操作为最后一张图片加载操作
        [blockOperation addDependency:lastBlockOperation];
        
        [operationQueue addOperation:blockOperation];
        
    }
    //将最后一个图片的加载操作加入线程队列
    [operationQueue addOperation:lastBlockOperation];
}
```

## GCD
GCD(Grand Central Dispatch)是基于C语言开发的一套多线程开发机制，也是目前苹果官方推荐的多线程开发方法。前面也说过三种开发中GCD抽象层次最高，当然是用起来也最简单，只是它基于C语言开发，并不像NSOperation是面向对象的开发，而是完全面向过程的。对于熟悉C#异步调用的朋友对于GCD学习起来应该很快，因为它与C#中的异步调用基本是一样的。这种机制相比较于前面两种多线程开发方式最显著的优点就是它对于多核运算更加有效。

GCD中也有一个类似于NSOperationQueue的队列，GCD统一管理整个队列中的任务。但是GCD中的队列分为并行队列和串行队列两类：

* 串行队列：只有一个线程，加入到队列中的操作按添加顺序依次执行。
* 并发队列：有多个线程，操作进来之后它会将这些队列安排在可用的处理器上，同时保证先进来的任务优先处理。

其实在GCD中还有一个特殊队列就是主队列，用来执行主线程上的操作任务（从前面的演示中可以看到其实在NSOperation中也有一个主队列）。

#### 串行队列

使用串行队列时首先要创建一个串行队列，然后调用异步调用方法，在此方法中传入串行队列和线程操作即可自动执行。下面使用线程队列演示图片的加载过程，你会发现多张图片会按顺序加载，因为当前队列中只有一个线程。


```objective-c
#pragma mark 多线程下载图片
-(void)loadImageWithMultiThread{
    int count=ROW_COUNT*COLUMN_COUNT;
    
    /*创建一个串行队列
     第一个参数：队列名称
     第二个参数：队列类型
    */
    dispatch_queue_t serialQueue=dispatch_queue_create("myThreadQueue1", DISPATCH_QUEUE_SERIAL);//注意queue对象不是指针类型
    //创建多个线程用于填充图片
    for (int i=0; i&lt;count; ++i) {
        //异步执行队列任务
        dispatch_async(serialQueue, ^{
            [self loadImage:[NSNumber numberWithInt:i]];
        });
        
    }
    //非ARC环境请释放
//    dispatch_release(seriQueue);
}
```

在上面的代码中更新UI还使用了GCD方法的主线程队列dispatch_get_main_queue()，其实这与前面两种主线程更新UI没有本质的区别。

#### 并发队列

并发队列同样是使用dispatch_queue_create()方法创建，只是最后一个参数指定为DISPATCH_QUEUE_CONCURRENT进行创建，但是在实际开发中我们通常不会重新创建一个并发队列而是使用dispatch_get_global_queue()方法取得一个全局的并发队列（当然如果有多个并发队列可以使用前者创建）。下面通过并行队列演示一下多个图片的加载。代码与上面串行队列加载类似，只需要修改照片加载方法如下：

```objective-c
-(void)loadImageWithMultiThread{
    int count=ROW_COUNT*COLUMN_COUNT;
    
    /*取得全局队列
     第一个参数：线程优先级
     第二个参数：标记参数，目前没有用，一般传入0
    */
    dispatch_queue_t globalQueue=dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    //创建多个线程用于填充图片
    for (int i=0; i&lt;count; ++i) {
        //异步执行队列任务
        dispatch_async(globalQueue, ^{
            [self loadImage:[NSNumber numberWithInt:i]];
        });
    }
}
```


![异步执行效果](image\8.png)

细心的朋友肯定会思考，既然可以使用dispatch_async()异步调用方法，是不是还有同步方法，确实如此，在GCD中还有一个dispatch_sync()方法。假设将上面的代码修改为同步调用，可以看到如下效果：

![同步执行效果](image\9.png)

可以看点击按钮后按钮无法再次点击，因为所有图片的加载全部在主线程中（可以打印线程查看），主线程被阻塞，造成图片最终是一次性显示。可以得出结论：

* 在GDC中一个操作是多线程执行还是单线程执行取决于当前队列类型和执行方法，只有队列类型为并行队列并且使用异步方法执行时才能在多个线程中执行。
* 串行队列可以按顺序执行，并行队列的异步方法无法确定执行顺序。
* UI界面的更新最好采用同步方法，其他操作采用异步方法。
* GCD中多线程操作方法不需要使用@autoreleasepool，GCD会管理内存。

#### 其他任务执行方法

GCD执行任务的方法并非只有简单的同步调用方法和异步调用方法，还有其他一些常用方法：

* dispatch_apply():重复执行某个任务，但是注意这个方法没有办法异步执行（为了不阻塞线程可以使用dispatch_async()包装一下再执行）。
* dispatch_once():单次执行一个任务，此方法中的任务只会执行一次，重复调用也没办法重复执行（单例模式中常用此方法）。
* dispatch_time()：延迟一定的时间后执行。
* dispatch_barrier_async()：使用此方法创建的任务首先会查看队列中有没有别的任务要执行，如果有，则会等待已有任务执行完毕再执行；同时在此方法后添加的任务必须等待此方法中任务执行后才能执行。（利用这个方法可以控制执行顺序，例如前面先加载最后一张图片的需求就可以先使用这个方法将最后一张图片加载的操作添加到队列，然后调用dispatch_async()添加其他图片加载任务）
* dispatch_group_async()：实现对任务分组管理，如果一组任务全部完成可以通过dispatch_group_notify()方法获得完成通知（需要定义dispatch_group_t作为分组标识）。


## 总结
* 1.无论使用哪种方法进行多线程开发，每个线程启动后并不一定立即执行相应的操作，具体什么时候由系统调度（CPU空闲时就会执行）。

* 2.更新UI应该在主线程（UI线程）中进行，并且推荐使用同步调用，常用的方法如下：

>-(void)performSelectorOnMainThread:(SEL)aSelector withObject:(id)arg waitUntilDone:(BOOL)wait

或者
> -(void)performSelector:(SEL)aSelector onThread:(NSThread *)thr withObject:(id)arg waitUntilDone:(BOOL) wait

方法传递主线程,[NSThread mainThread]

>[NSOperationQueue mainQueue] addOperationWithBlock:

使用dispatch_get_main_queue
>dispatch_sync(dispatch_get_main_queue(), ^{})

* 3.NSThread适合轻量级多线程开发，控制线程顺序比较难，同时线程总数无法控制（每次创建并不能重用之前的线程，只能创建一个新的线程）。

* 4.对于简单的多线程开发建议使用NSObject的扩展方法完成，而不必使用NSThread。

* 5.可以使用NSThread的currentThread方法取得当前线程，使用 sleepForTimeInterval:方法让当前线程休眠。

* 6.NSOperation进行多线程开发可以控制线程总数及线程依赖关系。

* 7.创建一个NSOperation不应该直接调用start方法（如果直接start则会在主线程中调用）而是应该放到NSOperationQueue中启动。

* 8.相比NSInvocationOperation推荐使用NSBlockOperation，代码简单，同时由于闭包性使它没有传参问题。

* 9.NSOperation是对GCD面向对象的ObjC封装，但是相比GCD基于C语言开发，效率却更高，建议如果任务之间有依赖关系或者想要监听任务完成状态的情况下优先选择NSOperation否则使用GCD。

* 10.在GCD中串行队列中的任务被安排到一个单一线程执行（不是主线程），可以方便地控制执行顺序；并发队列在多个线程中执行（前提是使用异步方法），顺序控制相对复杂，但是更高效。

* 11.在GDC中一个操作是多线程执行还是单线程执行取决于当前队列类型和执行方法，只有队列类型为并行队列并且使用异步方法执行时才能在多个线程中执行（如果是并行队列使用同步方法调用则会在主线程中执行）。

