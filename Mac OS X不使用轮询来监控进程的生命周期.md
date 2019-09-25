# Mac OS X不使用轮询来监控进程的生命周期

做mac应用开发与IOS一个很大的不同，是多进程，一个应用中存在多个的进程。很多时候我们都有监控进程的需求，多种OSX监控进程的方式，总有一种适合你。

## 简介
做一段时间的Mac OS X开发之后，你将不可避免的遇到需要创建协作进程的情况，例如：

* 在写一个应用程序中，你可能会想要将一些代码封装到一个独立的辅助进程中去。或许你想要将一些不可靠的代码放到一个独立的进程中去，这样，这个进程崩溃了也不会影响主程序的运行。又或者你可能想要访问某些不是线程安全的API，而不会锁定应用程序的用户界面。
* 您可能正在编写一套合作应用程序。也许你正在编写一个文字处理器，并且想要调用一个单独的公式编辑器服务。
* 如果您正在编写一个守护进程，则可能需要与可访问每个用户状态的各种代理程序进行交互。


一旦你有多个进程，就不可避免的遇到进程生命周期的问题：也就是说，一个进程需要知道另一个进程是否在运行。这篇文档描述了多种可以在进程启动或者终止时通知你的方法。它分为两个主要部分。监控一个你自己启动的进程，监控一个不是你自己启动的进程。最后，进程的序列号中包含了一些本文中讨论的进程序列号的基础API的重要信息。

首先，让我们谈论一个提供了大量关键优势的替代方法。


> **重要提示**：所有本文中讨论的技术都会在事件发生变化时通知你。通过轮询进程列表可以获得相同的信息，但轮询通常是一个坏主意（它消耗CPU时间，减少电池寿命，增加你设置进程的工作量，并且还会增加响应事件的延迟。）
## 面向服务的替代方案
监控进程生命周期的最常见原因之一是该进程为您提供一些服务。例如，一个电影转码应用程序，该程序通常会将实际的转码工作放到一个子进程中去执行，主进程负责监控子进程的工作状态，一旦子进程意外退出了，主进程可以重新启动它。

你可以通过重新构思你的方法来避免这个需求。与其明确的管理你的辅助进程状态，还不如将其重新定义为应用程序所需要的服务，然后通过launchd来管理该服务，它将负责启动和终止提供该服务的进程的所有细节。

关于面向服务的更全面的讨论，可以阅读launchd相关的文档。

## 监控自己启动的进程
有许多种不同的方式监控自己启动的进程，每种技术都各有利弊，阅读下面的内容，以选择一个最适合自己情况的。

### NSTask
NSTask可以轻松的启动一个帮助进程并等待它结束。你可以同步等待（使用-[NSTask waitUntilExit]方法），也可以注册一个通知，接收**NSTaskDidTerminateNotification**通知。代码清单1展示了同步方式，代码清单2展示了异步的方式。

**清单1：**同步使用NSTask
```
- (IBAction)testNSTaskSync:(id)sender
{
    NSTask *    syncTask;

    syncTask = [NSTask 
        launchedTaskWithLaunchPath:@"/bin/sleep" 
        arguments:[NSArray arrayWithObject:@"1"]
    ];
    [syncTask waitUntilExit];
}
```
**清单2：**异步使用NSTask
```
- (IBAction)testNSTaskAsync:(id)sender{
 task = [[NSTask alloc] init]; 
 [task setLaunchPath:@"/bin/sleep"];
 [task setArguments:[NSArray arrayWithObject:@"1"]];
 [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(taskExited:) name:NSTaskDidTerminateNotification object:task ];
 [task launch]; 
 //在下面的-taskExited：中继续执行。
}
- (void)taskExited:(NSNotification *)note{
 // 收到通知!
 [[NSNotificationCenter defaultCenter] removeObserver:self name:NSTaskDidTerminateNotification object:task ];
 [task release]; 
 task = nil;
}
```
### 进程死亡事件
如果您使用基于序列号的API启动应用程序，则可以通过注册kAEApplicationDiedApple事件来得知其终止。

> **重要提示**：此事件仅适用于您启动的应用程序。

清单3显示了如何注册和处理应用程序死亡事件。

**清单3：**使用进程死亡事件
```
- (IBAction)testApplicationDied:(id)sender
{
    NSURL *     url;
    static BOOL sHaveInstalledAppDiedHandler;

    if ( ! sHaveInstalledAppDiedHandler ) {
        (void) AEInstallEventHandler(
            kCoreEventClass, 
            kAEApplicationDied, 
            (AEEventHandlerUPP) AppDiedHandler, 
            (SRefCon) self, 
            false
        );
        sHaveInstalledAppDiedHandler = YES;
    }

    url = [NSURL fileURLWithPath:@"/Applications/TextEdit.app"];
    (void) LSOpenCFURLRef( (CFURLRef) url, NULL);

    // Execution continues in AppDiedHandler, below.
}

static OSErr AppDiedHandler(
    const AppleEvent *  theAppleEvent, 
    AppleEvent *        reply, 
    SRefCon             handlerRefcon
)
{
    SInt32              errFromEvent;
    ProcessSerialNumber psn;
    DescType            junkType;
    Size                junkSize;

    (void) AEGetParamPtr(
        theAppleEvent, 
        keyErrorNumber, 
        typeSInt32, 
        &junkType, 
        &errFromEvent, 
        sizeof(errFromEvent), 
        &junkSize
    );
    (void) AEGetParamPtr(
        theAppleEvent, 
        keyProcessSerialNumber, 
        typeProcessSerialNumber, 
        &junkType, 
        &psn, 
        sizeof(psn), 
        &junkSize
    );

    // You've been notified!

    NSLog(
        @"died %lu.%lu %d", 
        (unsigned long) psn.highLongOfPSN, 
        (unsigned long) psn.lowLongOfPSN, 
        (int) errFromEvent
    );

    return noErr;
}
```
> **重要提示：**进程死亡事件基于序列号，这是一个具有一些重要后果的事实。有关详细信息，请参阅过程序列号。

### UNIX方式
Mac OS X的BSD子系统有两个开启新进程的基本API：

* fork和exec—这种技术起源于第一个UNIX系统，使用fork创建一个进程，是对当前进程的精确克隆，而exec（实际上是一个基于exec的例程簇）会使当前进程去启动运行一个新的可执行文件。

* posix spawn—这个API就像fork和exec的组合。它是在Mac OS X 10.5中引入的。
  在上述两种情况下，生成的进程都是当前进程的子进程。有两种传统的UNIX方法能知道子进程的死亡：
* 同步方法：使用一系列的等待例程（典型的：waitpid）

* 异步方法：通过SIGCHLD信号(SIGCHLD，在一个进程终止或者停止时，将SIGCHLD信号发送给其父进程，按系统默认将忽略此信号，如果父进程希望被告知其子系统的这种状态，则应捕捉此信号。)

在许多情况下同步等待是比较适用的方法，例如，如果父进程在子进程完成之前无法进行，理所应当的应该同步等待。清单4显示了如何fork，然后exec ，再等待的示例。

**清单4：**Fork, exec, wait
```
extern char **environ;

- (IBAction)testWaitPID:(id)sender
{
    pid_t       pid;
    char *      args[3] = { "/bin/sleep", "1", NULL };
    pid_t       waitResult;
    int         status;

    // I used fork/exec rather than posix_spawn because I would like this 
    // code to be compatible with 10.4.x.

    pid = fork();
    switch (pid) {
        case 0:
            // child
            (void) execve(args[0], args, environ);
            _exit(EXIT_FAILURE);
            break;
        case -1:
            // error
            break;
        default:
            // parent
            break;
    }
    if (pid >= 0) {
        do {
            waitResult = waitpid(pid, &status, 0);
        } while ( (waitResult == -1) && (errno == EINTR) );
    }
}
```
另一方面，有些情况同步等待是一个非常糟糕的主意。例如，如果您正在应用程序的主线程上面运行，并且子进程可能会执行一个耗时操作，则不希望阻塞应用程序的用户界面等待该子进程退出。
在这种情况下，您可以通过监听SIGCHLD信号异步等待。
> **重要提示**：如果你使用监听SIGCHLD信号异步等待的方式，你仍然需要通过调用等待例程来获取子进程，否则将会导致僵尸进程。

由于与信号处理程序相关联的环境很复杂，可能监听信号会很棘手。具体来说，如果你使用了一个信号处理程序（[signal](x-man-page://3/signal)或[sigaction](x-man-page://2/sigaction)，那么你必须非常小心你在该处理程序中所做的工作。很少的函数能被信号处理程序安全的调用，例如，使用malloc分配内存空间就是不安全的。

内呗信号处理程序安全调用的函数（**async-signal safe**函数）列在[sigaction手册页上](x-man-page://2/sigaction)。
在大部分情况下，你必须采用额外的手段，将传入信号重定向到更加合理的环境中。有两种标准的做法：
* sockets（套接字）—在此技术中，您将创建一个UNIX域套接字对，并使用CFSocket将一端添加到您的循环中。当信号到达时，信号处理器将一个虚拟消息写入套接字。这将唤醒循环，并允许您在安全的环境中处理信号。要查看此技术的演示，请查看[示例代码“CFLocalServer”](https://developer.apple.com/samplecode/CFLocalServer/index.html)中的InstallSignalToSocket例程。
* kqueues —kqueue机制允许您收听信号而不安装任何信号处理程序。所以你可以创建一个kqueue，指示它来监听SIGCHLD信号，然后将其包装在一个CFFileDescriptor中并将其添加到你的runloop中。当信号到达时，与CFFileDescriptor关联的回调例程运行，您可以在安全的环境中处理信号。要查看此技术的演示，请查看[示例代码“PreLoginAgents”](https://developer.apple.com/samplecode/PreLoginAgents/index.html)中的InstallHandleSIGTERMFromRunLoop例程。
> **重要提示：** kqueue技术需要Mac OS X 10.5或更高版本，因为它使用CFFileDescriptor。

### UNIX替代方案
处理SIGCHLD信号有许多陷阱。上一节描述了最深刻的一部分，但还有其他部分。当你在写library code的时候，使用SIGCHLD是一件非常棘手的事情，因为SIGCHLD由主程序本身控制，你的library code不能要求其被设置为某种方式。
有多种方法来避免SIGCHLD的这种混乱，一种方式就是创建一个域套接字对，并且为了使子进程具有引用一端的唯一描述符，父进程具有另一端的描述符。当子进程中止的时候，系统关闭子进程的描述符，这导致套接字的另一端指向文件的结尾（这意味着它变成可读的了，但是当你尝试读取的时候，返回的是0）.当父进程监控到文件结束的状态，就可以获取子进程信息。清单5展示了这种方案的示例。

**清单5：**使用套接字监测子进程的中止
```
- (IBAction)testSocketPair:(id)sender
{
    int                 fds[2];
    int                 remoteSocket;
    int                 localSocket;
    CFSocketContext     context = { 0, self, NULL, NULL, NULL };
    CFRunLoopSourceRef  rls;
    char *              args[3] = { "/bin/sleep", "1", NULL } ;

    // Create a socket pair and wrap the local end up in a CFSocket.

    (void) socketpair(AF_UNIX, SOCK_STREAM, 0, fds);

    remoteSocket = fds[0];
    localSocket  = fds[1];
    socket = CFSocketCreateWithNative(
        NULL, 
        localSocket, 
        kCFSocketDataCallBack, 
        SocketClosedSocketCallBack, 
        &context
    );
    CFSocketSetSocketFlags(
        socket, 
        kCFSocketAutomaticallyReenableReadCallBack | kCFSocketCloseOnInvalidate
    );

    // Add the CFSocket to our runloop.

    rls = CFSocketCreateRunLoopSource(NULL, socket, 0);
    CFRunLoopAddSource(CFRunLoopGetCurrent(), rls, kCFRunLoopDefaultMode);
    CFRelease(rls);

    // fork and exec the child process.

    childPID = fork();
    switch (childPID) {
        case 0:
            // child
            (void) execve(args[0], args, environ);
            _exit(EXIT_FAILURE);
            break;
        case -1:
            // error
            break;
        default:
            // parent
            break;
    }

    // Close our reference to the remote socket. The only reference remaining 
    // is the one in the child. When that dies, the socket will become readable.

    (void) close(remoteSocket);

    // Execution continues in SocketClosedSocketCallBack, below.
}

static void SocketClosedSocketCallBack(
    CFSocketRef             s, 
    CFSocketCallBackType    type, 
    CFDataRef               address, 
    const void *            data, 
    void *                  info
)
{
    int             waitResult;
    int             status;

    // Reap the child.

    do {
        waitResult = waitpid( ((AppDelegate *) info)->childPID, &status, 0);
    } while ( (waitResult == -1) && (errno == EINTR) );

    // You've been notified!
}
```

## 监控任一进程
如果要监控一个非自己启动的进程，则可选用的方案比较少。但是，这些可以的方案就可以满足我们大部分的需求。此外，你要根据自己的情况选择正确的API，阅读以下部分以让你了解哪个API是适合你的。

### NSWorkspace
[NSWorkspace](http://developer.apple.com/documentation/Cocoa/Reference/ApplicationKit/Classes/NSWorkspace_Class/Reference/Reference.html)提供了一个非常简单的方式来监控进程的启动和退出。
要注册这些通知，您必须：

1、获得NSWorkspace的自定义通知中心 ，调用-[NSWorkspace notificationCenter]

2、添加NSWorkspaceDidLaunchApplicationNotification和NSWorkspaceDidTerminateApplicationNotification事件的观察者

当收到通知的时候，user info字典包含了受影响进程的信息。NSWorkspace.h头文件中列出了user info字典的key，以"NSApplicationPath"开头。
清单6显示了如何使用NSWorkspace获取应用程序启动和终止的示例。

**清单6：**使用NSWorkspace获取应用程序启动和终止
```
- (IBAction)testNSWorkspace:(id)sender
{
    NSNotificationCenter *  center;

    NSLog(@"-[AppDelegate testNSWorkspace:]");

    // Get the custom notification center.

    center = [[NSWorkspace sharedWorkspace] notificationCenter];

    // Install the notifications.

    [center addObserver:self 
        selector:@selector(appLaunched:) 
        name:NSWorkspaceDidLaunchApplicationNotification 
        object:nil
    ];
    [center addObserver:self 
        selector:@selector(appTerminated:) 
        name:NSWorkspaceDidTerminateApplicationNotification 
        object:nil
    ];

    // Execution continues in -appLaunched: and -appTerminated:, below.
}

- (void)appLaunched:(NSNotification *)note
{
    NSLog(@"launched %@\n", [[note userInfo] objectForKey:@"NSApplicationName"]);

    // You've been notified!
}

- (void)appTerminated:(NSNotification *)note
{
    NSLog(@"terminated %@\n", [[note userInfo] objectForKey:@"NSApplicationName"]);

    // You've been notified!
}
```

> **重要提示：** NSWorkspace基于序列号，这是一个具有一些重要后果的事实。有关详细信息，请参阅过程序列号

### Carbon Event Manager
[Carbon Event Manager](http://developer.apple.com/documentation/Carbon/Conceptual/Carbon_Event_Manager/index.html) 发送大量的与进程管理相关的事件，具体来说，当应用程序启动的时候，发送kEventAppLaunched事件，当应用程序中止时，发送kEventAppTerminated事件，你可以像任何其他Carbon事件一样注册这些事件。清单7显示了一个例子。

调用事件处理程序时，kEventParamProcessID参数将包含受影响的进程的ProcesSerialNumber。
> **重要提示：**当您的应用程序收到该kEventAppTerminated事件时，终止应用程序可能已经退出。因此，您无法获取有关该应用程序的信息GetProcessInformation。如果您需要有关终止应用程序的信息，则必须提前缓存。

**清单7：**使用Carbon事件来获取应用程序启动和终止
```
- (IBAction)testCarbonEvents:(id)sender
{
    static EventHandlerRef sCarbonEventsRef = NULL;
    static const EventTypeSpec kEvents[] = {
        { kEventClassApplication, kEventAppLaunched },
        { kEventClassApplication, kEventAppTerminated }
    };

    if (sCarbonEventsRef == NULL) {
        (void) InstallEventHandler(
            GetApplicationEventTarget(),
            (EventHandlerUPP) CarbonEventHandler,
            GetEventTypeCount(kEvents),
            kEvents,
            self,
            &sCarbonEventsRef
        );
    }

    // Execution continues in CarbonEventHandler, below.
}

static OSStatus CarbonEventHandler(
    EventHandlerCallRef inHandlerCallRef, 
    EventRef            inEvent, 
    void *              inUserData
)
{
    ProcessSerialNumber psn;

    (void) GetEventParameter(
        inEvent, 
        kEventParamProcessID, 
        typeProcessSerialNumber, 
        NULL, 
        sizeof(psn), 
        NULL, 
        &psn
    );
    switch ( GetEventKind(inEvent) ) {
        case kEventAppLaunched:
            NSLog(
                @"launched %u.%u", 
                (unsigned int) psn.highLongOfPSN, 
                (unsigned int) psn.lowLongOfPSN
            );
            // You've been notified!
            break;
        case kEventAppTerminated:
            NSLog(
                @"terminated %u.%u", 
                (unsigned int) psn.highLongOfPSN, 
                (unsigned int) psn.lowLongOfPSN
            );
            // You've been notified!
            break;
        default:
            assert(false);
    }
    return noErr;
}
```
### kqueues
NSWorkspace和Carbon事件只能在单个GUI登录上下文中工作。如果您正在编写一个不在GUI登录上下文中运行的程序（也许是守护程序），或者您需要监视与运行时不同的上下文中的进程，则需要考虑替代方法。 [kqueue](x-man-page://2/kqueue) NOTE_EXIT
 事件是一个不错的选择。您可以使用它来检测进程何时退出，无论它运行的是哪个上下文。与NSWorkspace和Carbon事件不同，您必须准确指定要监视的进程; 否则任何进程的中止都无法得到通知。
清单8是一个简单的例子，说明如何使用kqueue来监视特定进程的终止。

**清单8：**使用kqueue监视特定进程
```
static pid_t gTargetPID = -1;
    // We assume that some other code sets up gTargetPID.

- (IBAction)testNoteExit:(id)sender
{
    FILE *                  f;
    int                     kq;
    struct kevent           changes;
    CFFileDescriptorContext context = { 0, self, NULL, NULL, NULL };
    CFRunLoopSourceRef      rls;

    // Create the kqueue and set it up to watch for SIGCHLD. Use the 
    // new-in-10.5 EV_RECEIPT flag to ensure that we get what we expect.

    kq = kqueue();

    EV_SET(&changes, gTargetPID, EVFILT_PROC, EV_ADD | EV_RECEIPT, NOTE_EXIT, 0, NULL);
    (void) kevent(kq, &changes, 1, &changes, 1, NULL);

    // Wrap the kqueue in a CFFileDescriptor (new in Mac OS X 10.5!). Then 
    // create a run-loop source from the CFFileDescriptor and add that to the 
    // runloop.

    noteExitKQueueRef = CFFileDescriptorCreate(NULL, kq, true, NoteExitKQueueCallback, &context);
    rls = CFFileDescriptorCreateRunLoopSource(NULL, noteExitKQueueRef, 0);
    CFRunLoopAddSource(CFRunLoopGetCurrent(), rls, kCFRunLoopDefaultMode);
    CFRelease(rls);

    CFFileDescriptorEnableCallBacks(noteExitKQueueRef, kCFFileDescriptorReadCallBack);

    // Execution continues in NoteExitKQueueCallback, below.
}

static void NoteExitKQueueCallback(
    CFFileDescriptorRef f, 
    CFOptionFlags       callBackTypes, 
    void *              info
)
{
    struct kevent   event;

    (void) kevent( CFFileDescriptorGetNativeDescriptor(f), NULL, 0, &event, 1, NULL);

    NSLog(@"terminated %d", (int) (pid_t) event.ident);

    // You've been notified!
}
```

## 进程序列号（Process Serial Numbers）
Mac OS X具有许多用于进程管理的高级API，可以按进程序列号（ProcessSerialNumber）进行处理。这些包括启动服务，进程管理器和NSWorkspace。这些API都有三个重要的功能：

* 它们在单个GUI登录会话的上下文中工作。例如，如果您使用NSWorkspace来观察正在启动和终止的应用程序，那么只会在同一个GUI登录会话中运行的应用程序被通知。

* 他们只看到连接到窗口服务器的进程。
  例如，如果您使用NSTask来运行BSD命令行工具，如[find](https://developer.apple.com/library/content/technotes/tn2050/%3Cx-man-page://1/find%3E)，那么基于NSWorkspace的观察者将不会被通知该工具的启动或终止。

* 它们通常不能在GUI登录上下文之外运行的进程（例如，守护程序）使用

