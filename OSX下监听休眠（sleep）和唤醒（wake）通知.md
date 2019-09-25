

# OSX下监听休眠（sleep）和唤醒（wake）通知

## 前言
很多时候我们的客户端软件在电脑休眠之前都需要做一些事情，比如保存，清理工作等事件。这个时候我们就希望在电脑休眠前能够接收到通知，然后有足够的时间去处理这些事情，让我们的软件更好的运行，带来更好的用户体验。

## 正文
Cocoa 和 I/O Kit都可以用来接收休眠和唤醒通知，除了正常的接收通知以外，I/O Kit还可以阻止和延迟闲置休眠。注意，即使是I/O Kit，也无法阻止强制休眠，只能延迟强制休眠。


> **提示：**Mac OS X 有两种休眠模式- 强制休眠和闲置休眠.

* 当用户执行一些操作导致的电脑休眠是强制休眠。 盒上笔记本的盖子，从右上角的苹果菜单中选择休眠都会导致强制休眠. 在一些特定的情况下，系统也会触发强制休眠, 比如, 温度过高或电量过低.

* 在系统设置中节能下设置的时间长度下，电脑都没有被使用，则会进入闲置休眠。

**清单1：**在Cocoa下注册休眠和唤醒通知

```
- (void) receiveSleepNote: (NSNotification*) note
{
    NSLog(@"receiveSleepNote: %@", [note name]);
}
 
- (void) receiveWakeNote: (NSNotification*) note
{
    NSLog(@"receiveWakeNote: %@", [note name]);
}
 
- (void) fileNotifications
{
    //These notifications are filed on NSWorkspace's notification center, not the default
    // notification center. You will not receive sleep/wake notifications if you file
    //with the default notification center.
    [[[NSWorkspace sharedWorkspace] notificationCenter] addObserver: self
            selector: @selector(receiveSleepNote:)
            name: NSWorkspaceWillSleepNotification object: NULL];
 
    [[[NSWorkspace sharedWorkspace] notificationCenter] addObserver: self
            selector: @selector(receiveWakeNote:)
            name: NSWorkspaceDidWakeNotification object: NULL];
}
```

> **提示：**IOPMAssertionCreateWithName 是一个在Mac OS X 10.6雪豹系统下新增的API。IOPMAssertionCreateWithName 允许应用弹出一个带有简要说明的提示框，告诉用户为什么要阻止休眠。如果你想要支持早期的Mac OS X 版本，你可能会使用到基于清单3和4的API，或者已经被启动的API——IOPMAssertionCreate 。


**清单2：**在Mac OS X 10.6中使用I/O Kit阻止休眠

```
...
 
#import <IOKit/pwr_mgt/IOPMLib.h>
 
...
// kIOPMAssertionTypeNoDisplaySleep prevents display sleep,
// kIOPMAssertionTypeNoIdleSleep prevents idle sleep
 
//reasonForActivity is a descriptive string used by the system whenever it needs
//  to tell the user why the system is not sleeping. For example,
//  "Mail Compacting Mailboxes" would be a useful string.
 
//  NOTE: IOPMAssertionCreateWithName limits the string to 128 characters.
CFStringRef* reasonForActivity= CFSTR("Describe Activity Type");
 
IOPMAssertionID assertionID;
IOReturn success = IOPMAssertionCreateWithName(kIOPMAssertionTypeNoDisplaySleep,
                                    kIOPMAssertionLevelOn, reasonForActivity, &assertionID);
if (success == kIOReturnSuccess)
{
 
    //Add the work you need to do without
    //  the system sleeping here.
 
    success = IOPMAssertionRelease(assertionID);
    //The system will be able to sleep again.
}
...
```
**清单3：**在 I/O Kit下注册休眠和唤醒通知

```
#include <ctype.h>
#include <stdlib.h>
#include <stdio.h>
 
#include <mach/mach_port.h>
#include <mach/mach_interface.h>
#include <mach/mach_init.h>
 
#include <IOKit/pwr_mgt/IOPMLib.h>
#include <IOKit/IOMessage.h>
 
io_connect_t  root_port; // a reference to the Root Power Domain IOService
 
void
MySleepCallBack( void * refCon, io_service_t service, natural_t messageType, void * messageArgument )
{
    printf( "messageType %08lx, arg %08lx\n",
        (long unsigned int)messageType,
        (long unsigned int)messageArgument );
 
    switch ( messageType )
    {
 
        case kIOMessageCanSystemSleep:
            /* Idle sleep is about to kick in. This message will not be sent for forced sleep.
                Applications have a chance to prevent sleep by calling IOCancelPowerChange.
                Most applications should not prevent idle sleep.
 
                Power Management waits up to 30 seconds for you to either allow or deny idle
                sleep. If you don't acknowledge this power change by calling either
                IOAllowPowerChange or IOCancelPowerChange, the system will wait 30
                seconds then go to sleep.
            */
 
            //Uncomment to cancel idle sleep
            //IOCancelPowerChange( root_port, (long)messageArgument );
            // we will allow idle sleep
            IOAllowPowerChange( root_port, (long)messageArgument );
            break;
 
        case kIOMessageSystemWillSleep:
            /* The system WILL go to sleep. If you do not call IOAllowPowerChange or
                IOCancelPowerChange to acknowledge this message, sleep will be
                delayed by 30 seconds.
 
                NOTE: If you call IOCancelPowerChange to deny sleep it returns
                kIOReturnSuccess, however the system WILL still go to sleep.
            */
 
            IOAllowPowerChange( root_port, (long)messageArgument );
            break;
 
        case kIOMessageSystemWillPowerOn:
            //System has started the wake up process...
            break;
 
        case kIOMessageSystemHasPoweredOn:
            //System has finished waking up...
        break;
 
        default:
            break;
 
    }
}
 
 
int main( int argc, char **argv )
{
    // notification port allocated by IORegisterForSystemPower
    IONotificationPortRef  notifyPortRef;
 
    // notifier object, used to deregister later
    io_object_t            notifierObject;
   // this parameter is passed to the callback
    void*                  refCon;
 
    // register to receive system sleep notifications
 
    root_port = IORegisterForSystemPower( refCon, &notifyPortRef, MySleepCallBack, &notifierObject );
    if ( root_port == 0 )
    {
        printf("IORegisterForSystemPower failed\n");
        return 1;
    }
 
    // add the notification port to the application runloop
    CFRunLoopAddSource( CFRunLoopGetCurrent(),
            IONotificationPortGetRunLoopSource(notifyPortRef), kCFRunLoopCommonModes );
 
    /* Start the run loop to receive sleep notifications. Don't call CFRunLoopRun if this code
        is running on the main thread of a Cocoa or Carbon application. Cocoa and Carbon
        manage the main thread's run loop for you as part of their event handling
        mechanisms.
    */
    CFRunLoopRun();
 
    //Not reached, CFRunLoopRun doesn't return in this case.
    return (0);
    
}
```
停止接收 I/O Kit 的休眠通知, 你需要从应用程序的runloop中删除事件源，并做一些清理工作。

**清单4：**移除 I/O Kit休眠/唤醒通知

```
...
    // we no longer want sleep notifications:
 
    // remove the sleep notification port from the application runloop
    CFRunLoopRemoveSource( CFRunLoopGetCurrent(),
                           IONotificationPortGetRunLoopSource(notifyPortRef),
                           kCFRunLoopCommonModes );
 
    // deregister for system sleep notifications
    IODeregisterForSystemPower( &notifierObject );
 
    // IORegisterForSystemPower implicitly opens the Root Power Domain IOService
    // so we close it here
    IOServiceClose( root_port );
 
    // destroy the notification port allocated by IORegisterForSystemPower
    IONotificationPortDestroy( notifyPortRef );
...

```

