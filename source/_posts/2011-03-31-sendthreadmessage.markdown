---
layout: post
title: "为什么没有SendThreadMessage呢？"
date: 2011-03-31 23:24
comments: true
categories: [windows]
---

最近忙公司的项目（或是毕设吧），发现很长时间没有总结了。是该换换脑子了。

“为什么没有SendThreadMessage呢？”这个问题，就来自自己平时实现的一些程序逻辑中。在一些具体的场景中，对像我这样的初学者来说，往往喜欢通过windwos的消息机制来完成UI线程和worker线程之间的同步，而不是去通过信号量或其他的去做。所以，这个问题一直困惑了自己很久。而现在，就来搞明白这个、

google一下，这个问题，在一个大牛（Raymond Chen）http://blogs.msdn.com/b/oldnewthing/archive/2008/12/23/9248851.aspx）的博客中提到了，而且也引发了很多讨论。我这里简单的”翻译”一下Raymond Chen自己的看法。

”想象中的SendThreadMessage是如何工作的呢？调用SendMessage 把消息直接分发给窗口过程？但是我们没有看到消息泵，想象中的SendThreadMessage将会把消息分发给谁呢？因为我们没有‘thread window procedure’这样的东东去处理我们的消息。

是的，我们可以自己在我们的线程中做一个消息泵，但是，想象中的SendThreadMessage，需要等待这个消息处理完毕。但是，我们怎么能够知道这个消息处理完毕了？因为我们不可能等待DispatchMessage返回，而DispatchMessage失败则是因为我们并不知道应该往哪一个窗口分发消息。window manager给线程发送一个消息，仅此而已。

你可能会认为，我们可以等待知道下一个GetMessage or PeekMessage，这样我们可以确定这个消息解决了。但是，我们却不能保证下一个消息检索函数(GetMessage PeekMessage)，是来自我们之前的消息泵。比如，我们这个线程消息，启动了一个模态窗口，是的。当我们的消息检索函数告诉我们这个消息已经处理完毕了。但是，事实上那个模态窗口还在，因为他自己又创建了一个消息泵。“

这段虽然不长，但是却另我头大无比。GetMessage ，  DispatchMessage。这2个基本的函数，天天用，但是却对他们的行为知之甚少，算上第一次写HelloWorld 到现在，至少也有1年了，依然朦胧，感到十分惭愧。而这也就是这篇总结要做的。而这的确是一个庞大的工程，因为要了解这2个函数，需要把握windows的消息机制。而windwos 并没有给我们源代码参考，这里参考ReactOS的实现，虽然不是windows正统，但是，应该差不远，至少是和win2003的相似。开始步入正题。

我们首先需要了解的是，UI线程 和我们的普通的Worker线程之间的区别是什么。

msdn http://msdn.microsoft.com/en-us/library/ms644927提到：

”To avoid the overhead of creating a message queue for non–GUI threads, all threads are created initially without a message queue. The system creates a thread-specific message queue only when the thread makes its first call to one of the specific user functions; no GUI function calls result in the creation of a message queue.“

既然，系统创建每一个线程时都是普通的non–GUI thread，直到GDI， User函数调用，才为线程创建消息队列，那么我们就从这些函数调用开始。

windwos在开始时，和linux一样 图形这部分是在用户空间中的进程负责，后面为了减少进程之间的环境切换，而放入了内核中。那么在系统调用这层，我们就看到了有2种情况。一种调用是原来的”内核”的调用，而另一种是新加进来的原来在用户空间的调用，这部分被称为扩充系统调用，这部分代码被放在了可以动态安装的模块win32k.sys。与之对应，系统的调用表就有了2个，一个是只包括之前的”来自内核的系统调用“，另一个则在之前的基础上，增加了图形图像的系统调用。当我们的系统调用被发现是扩充系统调用时，也就是，原来的的表不能满足我们的要求。windwos会将会扩充系统调用表。并装载win32k.sys模块。那么，我们的普普通通的线程就开始变为GUI线程了。

激动人心的旅程就从这里开始了。

开源代码就是好，随意都能够贴出来。

	NTSTATUS
	NTAPI
	PsConvertToGuiThread(VOID)
	{
	    ULONG_PTR NewStack;
	    PVOID OldStack;
	    PETHREAD Thread = PsGetCurrentThread();
	    PEPROCESS Process = PsGetCurrentProcess();
	    NTSTATUS Status;
	    PAGED_CODE();
	 
	    /* Validate the previous mode */
	    if (KeGetPreviousMode() == KernelMode) return STATUS_INVALID_PARAMETER;
	 
	    /* If no win32k, crashes later */
	    ASSERT(PspW32ProcessCallout != NULL);
	 
	    /* Make sure win32k is here */
	    if (!PspW32ProcessCallout) return STATUS_ACCESS_DENIED;
	 
	    /* Make sure it's not already win32 */
	    if (Thread->Tcb.ServiceTable != KeServiceDescriptorTable)
	    {
	        /* We're already a win32 thread */
	        return STATUS_ALREADY_WIN32;
	    }
	 
	    /* Check if we don't already have a kernel-mode stack */
	    if (!Thread->Tcb.LargeStack)
	    {
	        /* We don't create one */
	        NewStack = (ULONG_PTR)MmCreateKernelStack(TRUE, 0);
	        if (!NewStack)
	        {
	            /* Panic in user-mode */
	            NtCurrentTeb()->LastErrorValue = ERROR_NOT_ENOUGH_MEMORY;
	            return STATUS_NO_MEMORY;
	        }
	 
	        /* We're about to switch stacks. Enter a guarded region */
	        KeEnterGuardedRegion();
	 
	        /* Switch stacks */
	        OldStack = KeSwitchKernelStack((PVOID)NewStack,
	                                       (PVOID)(NewStack - KERNEL_STACK_SIZE));
	 
	        /* Leave the guarded region */
	        KeLeaveGuardedRegion();
	 
	        /* Delete the old stack */
	        MmDeleteKernelStack(OldStack, FALSE);
	    }
	 
	    /* This check is bizare. Check out win32k later */
	    if (!Process->Win32Process)
	    {
	        /* Now tell win32k about us */
	        Status = PspW32ProcessCallout(Process, TRUE);
	        if (!NT_SUCCESS(Status)) return Status;
	    }
	 
	    /* Set the new service table */
	    Thread->Tcb.ServiceTable = KeServiceDescriptorTableShadow;
	    ASSERT(Thread->Tcb.Win32Thread == 0);
	 
	    /* Tell Win32k about our thread */
	    Status = PspW32ThreadCallout(Thread, PsW32ThreadCalloutInitialize);
	    if (!NT_SUCCESS(Status))
	    {
	        /* Revert our table */
	        Thread->Tcb.ServiceTable = KeServiceDescriptorTable;
	    }
	 
	    /* Return status */
	    return Status;
	}

之前没有提到的是，这里判断了一下线程system stack的大小，因为GUI线程要比普通的线程增加了更多的嵌套调用，从而需要更多的system stack。MmCreateKernelStack就是分配空间的函数。这里只是分配了64K的大小，普通的thread system stack大小为12K。当然，按照惯例，这里64K的堆栈，只是提交了其中12K的大小。并设置好guard page。超过12K则产生异常然后再分配空间。一个进程，如果有一个线程是GUI线程，那么这个进程就是GUI 进程，那么，如果不是GUI进程，我们当然先得把进程转过来。PspW32ProcessCallout是一个函数指针，指向Win32kProcessCallback。这里就是干这个了，会初始化一系列的结构体，键盘格式，GDI 句柄表等等。我们这里略过这些细节。

我们看到，系统的ServiceTable换成了大的表。而PspW32ThreadCallout指向Win32kThreadCallback，这里就完成了把普通线程转换成GUI线程的过程。对于操作系统这么复杂的东东来说，要初始化的结构体真是茫茫的多。我们这里关注一点，在Win32kThreadCallback中，我们找到了创建消息队列的入口。Win32Thread->MessageQueue = MsqCreateMessageQueue(Thread);

系统有了消息队列，但是，并不能构成真正的win32应用程序。我们开发者，还需要在自己的窗口程序中构造一个简单的Message Dump，让我们看看这个GetMessage，到底做了什么。

GetMessage，最后会调用NtUserGetMessage。

	BOOL APIENTRY
	NtUserGetMessage(PMSG pMsg,
	                  HWND hWnd,
	                  UINT MsgFilterMin,
	                  UINT MsgFilterMax )
	{
	    MSG Msg;
	    BOOL Ret;
	 
	    if ( (MsgFilterMin|MsgFilterMax) & ~WM_MAXIMUM )
	    {
	        EngSetLastError(ERROR_INVALID_PARAMETER);
	        return FALSE;
	    }
	 
	    UserEnterExclusive();
	 
	    RtlZeroMemory(&Msg, sizeof(MSG));
	 
	    Ret = co_IntGetPeekMessage(&Msg, hWnd, MsgFilterMin, MsgFilterMax, PM_REMOVE, TRUE);
	 
	    UserLeave();
	 
	    if (Ret)
	    {
	        _SEH2_TRY
	        {
	            ProbeForWrite(pMsg, sizeof(MSG), 1);
	            RtlCopyMemory(pMsg, &Msg, sizeof(MSG));
	        }
	        _SEH2_EXCEPT(EXCEPTION_EXECUTE_HANDLER)
	        {
	            SetLastNtError(_SEH2_GetExceptionCode());
	            Ret = FALSE;
	        }
	        _SEH2_END;
	    }
	 
	    return Ret;
	}

原谅我略过一些茫茫多的细节。

	BOOL FASTCALL
	co_IntGetPeekMessage( PMSG pMsg,
	                      HWND hWnd,
	                      UINT MsgFilterMin,
	                      UINT MsgFilterMax,
	                      UINT RemoveMsg,
	                      BOOL bGMSG )
	{
	    //.......
	    do
	    {
	        Present = co_IntPeekMessage( pMsg,
	                                     Window,
	                                     MsgFilterMin,
	                                     MsgFilterMax,
	                                     RemoveMsg,
	                                     bGMSG );
	        if (Present)
	        {
	           /* GetMessage or PostMessage must never get messages that contain pointers */
	           ASSERT(FindMsgMemory(pMsg->message) == NULL);
	 
	           if (pMsg->message != WM_PAINT && pMsg->message != WM_QUIT)
	           {
	              pti->timeLast = pMsg->time;
	              pti->ptLast   = pMsg->pt;
	           }
	 
	           // The WH_GETMESSAGE hook enables an application to monitor messages about to
	           // be returned by the GetMessage or PeekMessage function.
	 
	           co_HOOK_CallHooks( WH_GETMESSAGE, HC_ACTION, RemoveMsg & PM_REMOVE, (LPARAM)pMsg);
	 
	           if ( bGMSG )
	           {
	              Present = (WM_QUIT != pMsg->message);
	              break;
	           }
	        }
	 
	        if ( bGMSG )
	        {
	           if ( !co_IntWaitMessage(Window, MsgFilterMin, MsgFilterMax) )
	           {
	              Present = -1;
	              break;
	           }
	        }
	        else
	        {
	           if (!(RemoveMsg & PM_NOYIELD))
	           {
	              IdlePing();
	              // Yield this thread!
	              UserLeave();
	              ZwYieldExecution();
	              UserEnterExclusive();
	              // Fall through to exit.
	              IdlePong();
	           }
	           break;
	        }
	    }
	    while( bGMSG && !Present );
	 
	    // Been spinning, time to swap vinyl...
	    if (pti->pClientInfo->cSpins >= 100)
	    {
	       // Clear the spin cycle to fix the mix.
	       pti->pClientInfo->cSpins = 0;
	       //if (!(pti->TIF_flags & TIF_SPINNING)) FIXME need to swap vinyl..
	    }
	    return Present;
	}

IntGetPeekMessage，就是一个循环，不断的调用co_IntPeekMessage 从消息队列中取出消息，如果没有消息，那么我们就调用co_IntWaitMessage等待消息，然后往复，除非我们遇到了WM_QUIT。

co_IntPeekMessage 看来是实现的关键,而他也是PeekMessage的关键部分。同样,略过那些繁琐的细节。当然，这并不是指那些不重要，而是实在是太多了。这个函数是整个消息机制的核心部分。需要慢慢来。

说了这么多，我们还不知道消息队列是啥模样了。

	typedef struct _USER_MESSAGE_QUEUE
	{
	  /* Reference counter, only access this variable with interlocked functions! */
	  LONG References;
	 
	  /* Owner of the message queue */
	  struct _ETHREAD *Thread;
	  /* Queue of messages sent to the queue. */
	  LIST_ENTRY SentMessagesListHead;                          //被“发送”的消息队列
	  /* Queue of messages posted to the queue. */
	  LIST_ENTRY PostedMessagesListHead;                        //被"Post"的消息队列
	  /* Queue for hardware messages for the queue. */
	  LIST_ENTRY HardwareMessagesListHead;                      //来自硬件的消息队列
	 
	  //.........
	 
	  /* messages that are currently dispatched by other threads */
	  LIST_ENTRY DispatchingMessagesHead;                           //  已经发送而对方尚未处理的消息队列
	  /* messages that are currently dispatched by this message queue, required for cleanup */
	  LIST_ENTRY LocalDispatchingMessagesHead;                     // 本地正在分发的消息队列
	   
	  //........
	 
	} USER_MESSAGE_QUEUE, *PUSER_MESSAGE_QUEUE;

SentMessagesListHead 这个队列的东西是发送到我们这个消息队列的消息。 也就是，当其他地方调用SendMessage到我们这个消息队列时，那个消息会放在这个队列中。

PostedMessagesListHead 同理，是其他地方调用PostMessage，然后把他那个消息放在了这个队列中。

PostMessage这个函数比较容易实现，我们只需要挂在目标的PostedMessagesListHead队列中就可以了。但是SendMessage就要复杂很多了。

如果发送方和接收方是在一个线程中，那么SendMessage会直接调用本窗口的窗口过程函数来处理这个消息。

如果发送方和接收方不在一个线程中，那么发送方就必须要等待接收方的运行结果之后，才能继续执行。而这个，就形成了一个感觉上是同步的一个过程。感觉上这个似乎也不是很复杂。但也不是一个很简单的线程同步问题。

想一下这个问题，当GUI线程A向GUI线程B发送一个消息时，线程B处理A这个消息时，又需要向线程A发送一个消息。那么，这2个线程会死锁么？ 当然不会。要知道，windwos搞这一套为的就是构造一个完整的消息驱动机制，更抽象的讲，这个消息机制也算的上是一个线程通信机制。而这一套东东，最复杂的是在于，这些东东需要用户程序结合到一起，才能真正的运行起来。也就是说，我们的应用程序，必须符合windwos程序的规范，才能和windwos消息机制参与起来。而这个参与中最重要的东东就是我们之前提到的GetMessage，DispatchingMessagesHead 和 LocalDispatchingMessagesHead 则是实现这一套机制中非常重要的部分。

DispatchingMessagesHead  当我们自己SendMessage到其他地方时，我们的消息是需要等待对面的结果，那么这个需要等待的消息就被放置到这里。这里可能会对一些windwos菜鸟觉得困惑，困惑这个为什么能够形成一个队列呢？这里先把问题留下来。

让我们站在接受者的消息队列的角度来看，当有人给我们SendMessage了，我们需要在这里处理，也就是Message Dispatch，当我们搞出这个消息的返回值时，我们接受方，还必须等待对面的人把我们的这个消息的返回值拿走，这个消息才算是搞定了。这里由于可能是不同线程，甚至是不同进程之间数据传递。所以这些东西必须要考虑在内，而这些消息放在哪里呢？LocalDispatchingMessagesHead 就跳出来解决这个问题。

总的说一下，当我们SendMessage一个消息时，会挂在接收方的SentMessagesListHead队列中，并挂在发送方的DispatchingMessagesHead。

接受方先查看SentMessagesListHead 是否有消息，有的话，则从SendMessageListHead中删除掉，并添加到LocalDispatchingMessagesHead队列中，等我们把这个消息处理完毕，从LocalDispatchingMessagesHead将这个消息删除。

我们首先关注这4个队列。那个硬件的队列主要是鼠标和键盘的东东。

第一次看这个可能有点晕，不急。有一个笼统的概念之后，我们在来看细节。这部分还不是非常复杂。


	/*
	 * Internal version of PeekMessage() doing all the work
	 */
	BOOL FASTCALL
	co_IntPeekMessage( PMSG Msg,
	                   PWND Window,
	                   UINT MsgFilterMin,
	                   UINT MsgFilterMax,
	                   UINT RemoveMsg,
	                   BOOL bGMSG )
	{
	    //...
	    do
	    {
	        //..
	        /* Dispatch sent messages here. */
	        while ( co_MsqDispatchOneSentMessage(ThreadQueue) )
	        {
	           //...
	        }
	         
	        //...
	 
	        /* Now check for normal messages. */
	        if ((ProcessMask & QS_POSTMESSAGE) &&
	            MsqPeekMessage( ThreadQueue,
	                            RemoveMessages,
	                            Window,
	                            MsgFilterMin,
	                            MsgFilterMax,
	                            ProcessMask,
	                            Msg ))
	        {
	               return TRUE;
	        }
	 
	        /* Now look for a quit message. */
	        if (ThreadQueue->QuitPosted)
	        {
	            /* According to the PSDK, WM_QUIT messages are always returned, regardless
	               of the filter specified */
	            Msg->hwnd = NULL;
	            Msg->message = WM_QUIT;
	            Msg->wParam = ThreadQueue->QuitExitCode;
	            Msg->lParam = 0;
	            if (RemoveMessages)
	            {
	                ThreadQueue->QuitPosted = FALSE;
	                ClearMsgBitsMask(ThreadQueue, QS_POSTMESSAGE);
	                pti->pcti->fsWakeBits &= ~QS_ALLPOSTMESSAGE;
	                pti->pcti->fsChangeBits &= ~QS_ALLPOSTMESSAGE;
	            }
	            return TRUE;
	        }
	 
	        /* Check for hardware events. */
	        if ((ProcessMask & QS_MOUSE) &&
	            co_MsqPeekMouseMove( ThreadQueue,
	                                 RemoveMessages,
	                                 Window,
	                                 MsgFilterMin,
	                                 MsgFilterMax,
	                                 Msg ))
	        {
	            return TRUE;
	        }
	 
	        if ((ProcessMask & QS_INPUT) &&
	            co_MsqPeekHardwareMessage( ThreadQueue,
	                                       RemoveMessages,
	                                       Window,
	                                       MsgFilterMin,
	                                       MsgFilterMax,
	                                       ProcessMask,
	                                       Msg))
	        {
	            return TRUE;
	        }
	 
	        /* Check for sent messages again. */
	        while ( co_MsqDispatchOneSentMessage(ThreadQueue) )
	        {
	           if (HIWORD(RemoveMsg) && !bGMSG) Hit = TRUE;
	        }
	        if (Hit) return FALSE;
	 
	        /* Check for paint messages. */
	        if ((ProcessMask & QS_PAINT) &&
	            pti->cPaintsReady &&
	            IntGetPaintMessage( Window,
	                                MsgFilterMin,
	                                MsgFilterMax,
	                                pti,
	                                Msg,
	                                RemoveMessages))
	        {
	            return TRUE;
	        }
	 
	       /* This is correct, check for the current threads timers waiting to be
	          posted to this threads message queue. If any we loop again.
	        */
	        if ((ProcessMask & QS_TIMER) &&
	            PostTimerMessages(Window))
	        {
	            continue;
	        }
	 
	        return FALSE;
	    }
	    while (TRUE);
	 
	    return TRUE;
	}

co_MsqDispatchOneSentMessage 这里做的就是从SendMessageListHead 中取出一个别人SendMessage到我们这里的一个消息。 当我们把这些别人SendMessage给我们的消息处理完，就跳出那个循环，MsqPeekMessage 则去搞定别人PostMessage给我们的消息，最后再次检查一次co_MsqDispatchOneSentMessage，有没有人给我们发送了SendMessage消息，因为这之间的间隔是有可能有新的SendMessage消息。然后是IntGetPaintMessage 和PostTimerMessages这个名字就很容易理解了。而且，这里我们也看出了消息的优先级，为了提高Paint的效率，Paint是统一处理的。而且我们也看到了Timer消息，事实上我们看出他的优先级低于Paint，这样，我们就可以在timer中绘制函数，因为，我们每一次处理timer之前，我们能够保证我们的Paint消息已经被处理了。而且，我们也看出timer的确不准，在他前面有太多的东西要做了。

我们还需要了解下，我们的消息结构。是的，这个Post消息是要挂在队列中的。

	typedef struct _USER_MESSAGE
	{
	  LIST_ENTRY ListEntry;
	  MSG Msg;
	  DWORD QS_Flags;
	} USER_MESSAGE, *PUSER_MESSAGE;

Send的消息这里就要麻烦很多了。

	typedef struct _USER_SENT_MESSAGE
	{
	  LIST_ENTRY ListEntry;                            //接受方的队列
	  MSG Msg;
	  DWORD QS_Flags;  // Original QS bits used to create this message.
	  PKEVENT CompletionEvent;                    //这个用来做线程的唤醒操作
	  LRESULT* Result;
	  LRESULT lResult;
	  struct _USER_MESSAGE_QUEUE* SenderQueue;
	  struct _USER_MESSAGE_QUEUE* CallBackSenderQueue;
	  SENDASYNCPROC CompletionCallback;
	  ULONG_PTR CompletionCallbackContext;
	  /* entry in the dispatching list of the sender's message queue */
	  LIST_ENTRY DispatchingListEntry;                //发送方的DispatchingMessageList
	  INT HookMessage;
	  BOOL HasPackedLParam;
	} USER_SENT_MESSAGE, *PUSER_SENT_MESSAGE;

这个家伙，才是真正挂在发送队列中的数据结构，我们的MSG只是其中的一个数据成员。这里，就和我们之前提到的，这个消息，是在2个队列中存在，一边在发送方的DispatchingMessageList，表示这个消息正在分发，一边在接受方的SentMessagesListHead，表示这个消息被发送过来。等待处理。

让我们一看co_MsqDispatchOneSentMessage的究竟。

	BOOLEAN FASTCALL
	co_MsqDispatchOneSentMessage(PUSER_MESSAGE_QUEUE MessageQueue)
	{
	   PUSER_SENT_MESSAGE SaveMsg, Message;
	   PLIST_ENTRY Entry;
	   LRESULT Result;
	   PTHREADINFO pti;
	 
	   if (IsListEmpty(&MessageQueue->SentMessagesListHead))
	   {
	      return(FALSE);
	   }
	 
	   /* remove it from the list of pending messages */
	   Entry = RemoveHeadList(&MessageQueue->SentMessagesListHead);
	   Message = CONTAINING_RECORD(Entry, USER_SENT_MESSAGE, ListEntry);
	 
	   pti = MessageQueue->Thread->Tcb.Win32Thread;
	 
	   SaveMsg = pti->pusmCurrent;
	   pti->pusmCurrent = Message;
	 
	   // Processing a message sent to it from another thread.
	   if ( ( Message->SenderQueue && MessageQueue != Message->SenderQueue) ||
	        ( Message->CallBackSenderQueue && MessageQueue != Message->CallBackSenderQueue ))
	   {  // most likely, but, to be sure.
	      pti->pcti->CTI_flags |= CTI_INSENDMESSAGE; // Let the user know...
	   }
	 
	   /* insert it to the list of messages that are currently dispatched by this
	      message queue */
	   InsertTailList(&MessageQueue->LocalDispatchingMessagesHead,
	                  &Message->ListEntry);
	 
	   ClearMsgBitsMask(MessageQueue, Message->QS_Flags);
	 
	   if (Message->HookMessage == MSQ_ISHOOK)
	   {  // Direct Hook Call processor
	      Result = co_CallHook( Message->Msg.message,     // HookId
	                           (INT)(INT_PTR)Message->Msg.hwnd, // Code
	                            Message->Msg.wParam,
	                            Message->Msg.lParam);
	   }
	   else if (Message->HookMessage == MSQ_ISEVENT)
	   {  // Direct Event Call processor
	      Result = co_EVENT_CallEvents( Message->Msg.message,
	                                    Message->Msg.hwnd,
	                                    Message->Msg.wParam,
	                                    Message->Msg.lParam);
	   }
	   else
	   {  /* Call the window procedure. */
	      Result = co_IntSendMessage( Message->Msg.hwnd,
	                                  Message->Msg.message,
	                                  Message->Msg.wParam,
	                                  Message->Msg.lParam);
	   }
	 
	   /* remove the message from the local dispatching list, because it doesn't need
	      to be cleaned up on thread termination anymore */
	   RemoveEntryList(&Message->ListEntry);
	 
	   /* remove the message from the dispatching list if needed, so lock the sender's message queue */
	   if (!(Message->HookMessage & MSQ_SENTNOWAIT))
	   {
	      if (Message->DispatchingListEntry.Flink != NULL)
	      {
	         /* only remove it from the dispatching list if not already removed by a timeout */
	         RemoveEntryList(&Message->DispatchingListEntry);
	      }
	   }
	   /* still keep the sender's message queue locked, so the sender can't exit the
	      MsqSendMessage() function (if timed out) */
	 
	   if (Message->QS_Flags & QS_SMRESULT)
	   {
	      Result = Message->lResult;
	   }
	 
	   /* Let the sender know the result. */
	   if (Message->Result != NULL)
	   {
	      *Message->Result = Result;
	   }
	 
	   if (Message->HasPackedLParam == TRUE)
	   {
	      if (Message->Msg.lParam)
	         ExFreePool((PVOID)Message->Msg.lParam);
	   }
	 
	   /* Notify the sender. */
	   if (Message->CompletionEvent != NULL)
	   {
	      KeSetEvent(Message->CompletionEvent, IO_NO_INCREMENT, FALSE);
	   }
	 
	   /* Call the callback if the message was sent with SendMessageCallback */
	   if (Message->CompletionCallback != NULL)
	   {
	      co_IntCallSentMessageCallback(Message->CompletionCallback,
	                                    Message->Msg.hwnd,
	                                    Message->Msg.message,
	                                    Message->CompletionCallbackContext,
	                                    Result);
	   }
	 
	   /* Only if it is not a no wait message */
	   if (!(Message->HookMessage & MSQ_SENTNOWAIT))
	   {
	      IntDereferenceMessageQueue(Message->SenderQueue);
	      IntDereferenceMessageQueue(MessageQueue);
	   }
	 
	   /* free the message */
	   ExFreePoolWithTag(Message, TAG_USRMSG);
	 
	   /* do not hangup on the user if this is reentering */
	   if (!SaveMsg) pti->pcti->CTI_flags &= ~CTI_INSENDMESSAGE;
	   pti->pusmCurrent = SaveMsg;
	 
	   return(TRUE);
	}

我们首先从SentMessagesListHead把消息移动到LocalDispatchingMessagesHead，让我们略掉那些细节的标志位和hook的部分。co_IntSendMessage，则把这个消息发送出去，然后把结果给我们，然后我们把消息从接收方的LocalDispatchingMessagesHead，删掉。如果发送方还在等我们的消息，我们就把他从发送方的DispatchingMessagesHead中删掉这条消息，（因为有些消息，是有时间限制的，可能已经早就被从DispatchingMessagesHead删掉了）。然后把返回结果保存起来。当然，有些消息还是有附件的，一些资源需要释放。这里是那些消息就不在这里赘述了，而且我们也不关心这些。然后，我们通过Message->CompletionEvent来通知发送方，该醒过来了。最后，我们看到，如果这个消息有回调函数，这里并没有直接调用回调函数，而是又通过了消息机制发送了一个消息给自己（在自己的Post队列中）。有了这个，的确很容易去理解MSDN的相关意思了。有时候，真的。MS的文档为什么那么全，因为他不给我们看源代码，有源代码还需要那么多的详细文档么？而且，那些文档真的不能彻底说清楚。

转了这么远，问题又被迭代到co_IntSendMessage 上了。co_IntSendMessage 其实是co_IntSendMessageTimeout 的一个特殊调用。

	LRESULT FASTCALL
	co_IntSendMessageTimeout( HWND hWnd,
	                          UINT Msg,
	                          WPARAM wParam,
	                          LPARAM lParam,
	                          UINT uFlags,
	                          UINT uTimeout,
	                          ULONG_PTR *uResult )
	{
	    PWND DesktopWindow;
	    HWND *Children;
	    HWND *Child;
	 
	    if (HWND_BROADCAST != hWnd)
	    {
	        return co_IntSendMessageTimeoutSingle(hWnd, Msg, wParam, lParam, uFlags, uTimeout, uResult);
	    }
	 
	    DesktopWindow = UserGetWindowObject(IntGetDesktopWindow());
	    if (NULL == DesktopWindow)
	    {
	        EngSetLastError(ERROR_INTERNAL_ERROR);
	        return 0;
	    }
	 
	    /* Send message to the desktop window too! */
	    co_IntSendMessageTimeoutSingle(DesktopWindow->head.h, Msg, wParam, lParam, uFlags, uTimeout, uResult);
	 
	    Children = IntWinListChildren(DesktopWindow);
	    if (NULL == Children)
	    {
	        return 0;
	    }
	 
	    for (Child = Children; NULL != *Child; Child++)
	    {
	        co_IntSendMessageTimeoutSingle(*Child, Msg, wParam, lParam, uFlags, uTimeout, uResult);
	    }
	 
	    ExFreePool(Children);
	 
	    return (LRESULT) TRUE;
	}

我们不考虑广播的情况，看简单的给单个窗口发送消息的co_IntSendMessageTimeoutSingle

	static LRESULT FASTCALL
	co_IntSendMessageTimeoutSingle( HWND hWnd,
	                                UINT Msg,
	                                WPARAM wParam,
	                                LPARAM lParam,
	                                UINT uFlags,
	                                UINT uTimeout,
	                                ULONG_PTR *uResult )
	{
	    NTSTATUS Status;
	    PWND Window = NULL;
	    PMSGMEMORY MsgMemoryEntry;
	    INT lParamBufferSize;
	    LPARAM lParamPacked;
	    PTHREADINFO Win32Thread;
	    ULONG_PTR Result = 0;
	    DECLARE_RETURN(LRESULT);
	    USER_REFERENCE_ENTRY Ref;
	 
	    if (!(Window = UserGetWindowObject(hWnd)))
	    {
	        RETURN( FALSE);
	    }
	 
	    UserRefObjectCo(Window, &Ref);
	 
	    Win32Thread = PsGetCurrentThreadWin32Thread();
	 
	    IntCallWndProc( Window, hWnd, Msg, wParam, lParam);
	 
	    if ( NULL != Win32Thread &&
	         Window->head.pti->MessageQueue == Win32Thread->MessageQueue)
	    {
	        //本线程的消息，我们直接调用用户的窗口回调函数，终于要结束了。
	        Result = (ULONG_PTR)co_IntCallWindowProc( Window->lpfnWndProc,
	                                                  !Window->Unicode,
	                                                  hWnd,
	                                                  Msg,
	                                                  wParam,
	                                                  lParamPacked,
	                                                  lParamBufferSize );
	        if(uResult)
	        {
	            *uResult = Result;
	        }
	 
	        ObDereferenceObject(Win32Thread->pEThread);
	 
	        IntCallWndProcRet( Window, hWnd, Msg, wParam, lParam, (LRESULT *)uResult);
	 
	        if (! NT_SUCCESS(UnpackParam(lParamPacked, Msg, wParam, lParam, FALSE)))
	        {
	            DPRINT1("Failed to unpack message parameters\n");
	            RETURN( TRUE);
	        }
	 
	        RETURN( TRUE);
	    }
	 
	    //不是本线程，我们只能去转发这个消息了。
	 
	    do
	    {
	        Status = co_MsqSendMessage( Window->head.pti->MessageQueue,
	                                    hWnd,
	                                    Msg,
	                                    wParam,
	                                    lParam,
	                                    uTimeout,
	                                    (uFlags & SMTO_BLOCK),
	                                    MSQ_NORMAL,
	                                    uResult );
	    }
	    while ((STATUS_TIMEOUT == Status) &&
	           (uFlags & SMTO_NOTIMEOUTIFNOTHUNG) &&
	           !MsqIsHung(Window->head.pti->MessageQueue));
	 
	    IntCallWndProcRet( Window, hWnd, Msg, wParam, lParam, (LRESULT *)uResult);
	 
	    if (STATUS_TIMEOUT == Status)
	    {
	        /*
	MSDN says:
	    Microsoft Windows 2000: If GetLastError returns zero, then the function
	    timed out.
	    XP+ : If the function fails or times out, the return value is zero.
	    To get extended error information, call GetLastError. If GetLastError
	    returns ERROR_TIMEOUT, then the function timed out.
	*/
	        EngSetLastError(ERROR_TIMEOUT);
	        RETURN( FALSE);
	    }
	    else if (! NT_SUCCESS(Status))
	    {
	        SetLastNtError(Status);
	        RETURN( FALSE);
	    }
	 
	    RETURN( TRUE);
	 
	CLEANUP:
	    if (Window) UserDerefObjectCo(Window);
	    END_CLEANUP;
	}

这里我们终于看到结果了。当然，这里又给我们带出一个问题”系统是如何调用我们写的函数呢？是在什么时候调用？是通过什么方式？”这同样是，特别是第一次写windwos程序的菜鸟们遇到的第一个问题。这个问题说清楚还是挺麻烦的。这部分这里先留下。

让我们把大脑堆栈弹到开始。

还是这个问题”系统是如何调用我们写的函数呢？是在什么时候调用？是通过什么方式？”现在我们还不能回答所有问题，但是却可以回答”系统什么时候调用我们的窗口过程函数”。

我们调用系统的代码，或是说是调用系统服务，API等什么的，是通过中断机制完成的。并通过查找系统调用表来找到相对应的系统函数。也就是，我们可以随时随地利用中断机制去执行系统代码（当然是在限制下）。那么，系统可以随时随地的去执行我们用户空间的代码么？有点难，我们不去思考那么复杂的，因为还有一些其他的机制做这些类似的工作。我们只是去思考其中的一种，如何调用我们的窗口过程函数。

很容易想到，随时随地执行用户的代码很难。因为没有硬件的支持去让我们完成类似中断的机制。那系统只能在一些特定的地方才能有机会去执行我们的窗口过程函数。显然，GetMessage就是这个执行用户窗口过程函数的地方。而当用户程序在处理一个消息时，系统是没有办法有任何作为的。只能等待用户下一次调用GetMessage类似的函数，才能重新获得代码的控制。我们在co_IntPeekMessage中看出些端倪。如果消息队列中，没有任何消息，那么GetMessage并不会退出，也就是不将执行权给用户的代码，而是进入等待状态。如果这时来的一些SendMessage的消息，线程会唤醒并执行这些代码。除非有一个Post或是其他消息，才会从GetMessage返回给用户空间。

换句话就是，如果我们的Sendmessage是发给不同的线程，只能在GetMessage这个函数内部执行。如果那个接收方的线程阻塞了，那么我们的SendMessage就不会返回，因为他并没有执行GetMessage。

在去思考另一个问题，当我们Sendmessage到另一个线程，而另一个线程并没有执行我们的GetMessage，在执行他的代码，而我们的线程看起来显然是被挂起等待了，是么？并不是，因为他还是可以接受其他线程发送过来的消息。这显然是处理在处理我们之前讨论过的一种情况。的确很有意思。因为从windwos的角度看，需要实现这种强壮的消息机制。那么这是一个什么过程呢？清楚一点。其实就是需要一种机制，也就是在等待对方线程处理完毕之前，可以处理别人发给我们的消息。哈哈。WaitForMultipleObjects等待2个event一个是要等待处理完毕的消息，一个是要等待sendmessage过来的新消息。当醒来时判断是什么让我们清醒过来，如果对面的线程不给力，我们只能继续循环等待。而这个也就是sendmessage的过程。

	NTSTATUS FASTCALL
	co_MsqSendMessage(PUSER_MESSAGE_QUEUE MessageQueue,
	                  HWND Wnd, UINT Msg, WPARAM wParam, LPARAM lParam,
	                  UINT uTimeout, BOOL Block, INT HookMessage,
	                  ULONG_PTR *uResult)
	{
	   PTHREADINFO pti;
	   PUSER_SENT_MESSAGE Message;
	   KEVENT CompletionEvent;
	   NTSTATUS WaitStatus;
	   PUSER_MESSAGE_QUEUE ThreadQueue;
	   LARGE_INTEGER Timeout;
	   PLIST_ENTRY Entry;
	   LRESULT Result = 0;   //// Result could be trashed. ////
	 
	   if(!(Message = ExAllocatePoolWithTag(PagedPool, sizeof(USER_SENT_MESSAGE), TAG_USRMSG)))
	   {
	      DPRINT1("MsqSendMessage(): Not enough memory to allocate a message");
	      return STATUS_INSUFFICIENT_RESOURCES;
	   }
	 
	   KeInitializeEvent(&CompletionEvent, NotificationEvent, FALSE);
	 
	   pti = PsGetCurrentThreadWin32Thread();
	   ThreadQueue = pti->MessageQueue;
	   ASSERT(ThreadQueue != MessageQueue);
	 
	   Timeout.QuadPart = (LONGLONG) uTimeout * (LONGLONG) -10000;
	 
	   /* FIXME - increase reference counter of sender's message queue here */
	 
	   Message->Msg.hwnd = Wnd;
	   Message->Msg.message = Msg;
	   Message->Msg.wParam = wParam;
	   Message->Msg.lParam = lParam;
	   Message->CompletionEvent = &CompletionEvent;
	   Message->Result = &Result;
	   Message->lResult = 0;
	   Message->QS_Flags = 0;
	   Message->SenderQueue = ThreadQueue;
	   Message->CallBackSenderQueue = NULL;
	   IntReferenceMessageQueue(ThreadQueue);
	   Message->CompletionCallback = NULL;
	   Message->CompletionCallbackContext = 0;
	   Message->HookMessage = HookMessage;
	   Message->HasPackedLParam = FALSE;
	 
	   IntReferenceMessageQueue(MessageQueue);
	 
	   /* add it to the list of pending messages */
	   InsertTailList(&ThreadQueue->DispatchingMessagesHead, &Message->DispatchingListEntry);
	 
	   /* queue it in the destination's message queue */
	   InsertTailList(&MessageQueue->SentMessagesListHead, &Message->ListEntry);
	 
	   Message->QS_Flags = QS_SENDMESSAGE;
	   MsqWakeQueue(MessageQueue, QS_SENDMESSAGE, TRUE);
	 
	   /* we can't access the Message anymore since it could have already been deleted! */
	 
	   if(Block)
	   {
	      //我们绝大部分都是不阻塞的。
	   }
	   else
	   {
	      PVOID WaitObjects[2];
	 
	      WaitObjects[0] = &CompletionEvent;
	      WaitObjects[1] = ThreadQueue->NewMessages;
	      do
	      {
	         UserLeaveCo();
	 
	         WaitStatus = KeWaitForMultipleObjects(2, WaitObjects, WaitAny, UserRequest,
	                                               UserMode, FALSE, (uTimeout ? &Timeout : NULL), NULL);
	 
	         UserEnterCo();
	 
	         if(WaitStatus == STATUS_TIMEOUT)
	         {
	            //...
	         }
	         while (co_MsqDispatchOneSentMessage(ThreadQueue))
	            ;
	      }
	      while (NT_SUCCESS(WaitStatus) && STATUS_WAIT_0 != WaitStatus);
	   }
	 
	   if(WaitStatus != STATUS_TIMEOUT)
	      *uResult = (STATUS_WAIT_0 == WaitStatus ? Result : -1);
	 
	   return WaitStatus;
	}

GetMessage返回了，一般是跑2个函数。

TranslateMessage(&msg); 
DispatchMessage(&msg);

这里我们不讨论TranslateMessage，这个主要是辅助一些硬件消息相关。

DispatchMessage的事情，就是做这个调用相对用的窗口过程部分。这部分主要是从系统调用我们的代码，目前对这个还没有什么兴趣。

类似的还有模态窗口，产生模态窗口的窗口，会阻塞一些消息，但是却不是阻塞所有的消息，别的线程依然可以给发SendMessage。为什么呢？他们之间会有联系么？
