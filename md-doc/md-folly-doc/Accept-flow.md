#Accept流程

* folly::EventHandler::registerImpl 
* 	folly::EventHandler::registerHandler
*	folly::AsyncServerSocket::startAccepting 
* folly::AsyncServerSocket::<lambda()>::operator()(void) const 
		/data2/CppSrc/projects/folly/folly/io/async/AsyncServerSocket.cpp:657
 folly::detail::ScopeGuardForNewException<folly::AsyncServerSocket::addAcceptCallback(folly::AsyncServerSocket::AcceptCallback*, folly::EventBase*, uint32_t)::<lambda()>, false>::~ScopeGuardForNewException(void) 
*  folly::AsyncServerSocket::addAcceptCallback
* wangle::AsyncServerSocketFactory::addAcceptCB




EventHandler::libeventCallback就是回调了！

~~~cpp

bool EventHandler::registerImpl ( uint16_t events, bool internal )
{
    struct event_base* evb = event_.ev_base;
    event_set ( &event_, event_.ev_fd, events,
                &EventHandler::libeventCallback, this );
    event_base_set ( evb, &event_ );

    // Set EVLIST_INTERNAL if this is an internal event
    if ( internal )
    {
        event_ref_flags ( &event_ ) |= EVLIST_INTERNAL;
    }

    if ( event_add ( &event_, nullptr ) < 0 )
    {
        LOG ( ERROR ) << "EventBase: failed to register event handler for fd "
                      << event_.ev_fd << ": " << strerror ( errno );
        // Call event_del() to make sure the event is completely uninstalled
        event_del ( &event_ );
        return false;
    }

    return true;
}
~~~


AsyncServerSocket::handlerReady里面有限流逻辑！
folly::EventHandler::libeventCallback 
folly::AsyncServerSocket::handlerReady
folly::AsyncServerSocket::dispatchSocket


dispatch的话就弄到了IOExecutor的Queun里面去了？

(gdb) bt
#0  boost::variant<folly::IOBuf*, folly::AsyncTransportWrapper*, wangle::ConnInfo&, wangle::ConnEvent, std::tuple<folly::IOBuf*, std::shared_ptr<folly::AsyncUDPSocket>, folly::SocketAddress> >::variant<wangle::ConnInfo> (this=0x7fffefffde20, operand=...) at /data2/CppSrc/projects/boost_1_63_0/boost/variant/variant.hpp:1776
#1  0x00000000008dc453 in wangle::ServerAcceptor<wangle::Pipeline<folly::IOBufQueue&, std::string> >::onNewConnection (this=0xe63b00, transport=std::unique_ptr<folly::AsyncTransportWrapper> containing 0x0, clientAddr=0x7fffefffe420, nextProtocolName="", secureTransportType=wangle::SecureTransportType::NONE, tinfo=...) at /data2/CppSrc/projects/wangle/wangle/bootstrap/ServerBootstrap-inl.h:207
#2  0x00000000006fcd3d in wangle::Acceptor::connectionReady (this=0xe63b00, sock=std::unique_ptr<folly::AsyncTransportWrapper> containing 0x0, clientAddr=..., nextProtocolName="", secureTransportType=wangle::SecureTransportType::NONE, tinfo=...) at /data2/CppSrc/projects/wangle/wangle/acceptor/Acceptor.cpp:308
#3  0x00000000006fcdd2 in wangle::Acceptor::plaintextConnectionReady (this=0xe63b00, sock=std::unique_ptr<folly::AsyncTransportWrapper> containing 0x0, clientAddr=..., nextProtocolName="", secureTransportType=wangle::SecureTransportType::NONE, tinfo=...) at /data2/CppSrc/projects/wangle/wangle/acceptor/Acceptor.cpp:323
#4  0x00000000006fcb83 in wangle::Acceptor::processEstablishedConnection (this=0xe63b00, fd=29, clientAddr=..., acceptTime=..., tinfo=...) at /data2/CppSrc/projects/wangle/wangle/acceptor/Acceptor.cpp:273
#5  0x00000000006fc5b2 in wangle::Acceptor::onDoneAcceptingConnection (this=0xe63b00, fd=29, clientAddr=..., acceptTime=...) at /data2/CppSrc/projects/wangle/wangle/acceptor/Acceptor.cpp:224
#6  0x00000000006fc54e in wangle::Acceptor::connectionAccepted (this=0xe63b00, fd=29, clientAddr=...) at /data2/CppSrc/projects/wangle/wangle/acceptor/Acceptor.cpp:216
#7  0x00000000005569e7 in folly::AsyncServerSocket::RemoteAcceptor::messageAvailable(folly::AsyncServerSocket::QueueMessage&&) (this=0x7ffff0001aa0, msg=<unknown type in /data2/CppSrc/projects/cmake-build-debug/wangleserver, CU 0x667e75, DIE 0x6990f0>) at /data2/CppSrc/projects/folly/folly/io/async/AsyncServerSocket.cpp:115
#8  0x0000000000564b6a in folly::NotificationQueue<folly::AsyncServerSocket::QueueMessage>::Consumer::consumeMessages (this=0x7ffff0001aa0, isDrain=false, numConsumed=0x0) at /data2/CppSrc/projects/folly/folly/io/async/NotificationQueue.h:868
#9  0x000000000056461a in folly::NotificationQueue<folly::AsyncServerSocket::QueueMessage>::Consumer::handlerReady (this=0x7ffff0001aa0) at /data2/CppSrc/projects/folly/folly/io/async/NotificationQueue.h:794
#10 0x00000000005a00fb in folly::EventHandler::libeventCallback (fd=25, events=2, arg=0x7ffff0001ab0) at /data2/CppSrc/projects/folly/folly/io/async/EventHandler.cpp:184
#11 0x000000000042c0fa in event_persist_closure (ev=<optimized out>, base=0x7fffe8000c80) at /data2/CppSrc/projects/libevent-2.1.8-stable/event.c:1580
#12 event_process_active_single_queue (base=base@entry=0x7fffe8000c80, activeq=0x7fffe80010d0, max_to_process=max_to_process@entry=2147483647, endtime=endtime@entry=0x0) at /data2/CppSrc/projects/libevent-2.1.8-stable/event.c:1639
#13 0x000000000042cb8f in event_process_active (base=0x7fffe8000c80) at /data2/CppSrc/projects/libevent-2.1.8-stable/event.c:1738
#14 event_base_loop (base=0x7fffe8000c80, flags=1) at /data2/CppSrc/projects/libevent-2.1.8-stable/event.c:1961
#15 0x000000000058e061 in folly::EventBase::loopBody (this=0x7fffe80009f0, flags=0) at /data2/CppSrc/projects/folly/folly/io/async/EventBase.cpp:345
#16 0x000000000058dbc3 in folly::EventBase::loop (this=0x7fffe80009f0) at /data2/CppSrc/projects/folly/folly/io/async/EventBase.cpp:276
#17 0x000000000058f0a9 in folly::EventBase::loopForever (this=0x7fffe80009f0) at /data2/CppSrc/projects/folly/folly/io/async/EventBase.cpp:505
#18 0x000000000074e0a2 in wangle::IOThreadPoolExecutor::threadRun (this=0xe62160, thread=std::shared_ptr (count 5, weak 0) 0xe62780) at /data2/CppSrc/projects/wangle/wangle/concurrent/IOThreadPoolExecutor.cpp:156
#19 0x000000000075d3ab in std::_Mem_fn_base<void (wangle::ThreadPoolExecutor::*)(std::shared_ptr<wangle::ThreadPoolExecutor::Thread>), true>::operator()<std::shared_ptr<wangle::ThreadPoolExecutor::Thread>&, void> (this=0xe62b40, __object=0xe62160) at /opt/rh/devtoolset-4/root/usr/include/c++/5.3.1/functional:600
#20 0x000000000075bdf1 in std::_Bind<std::_Mem_fn<void (wangle::ThreadPoolExecutor::*)(std::shared_ptr<wangle::ThreadPoolExecutor::Thread>)> (wangle::ThreadPoolExecutor*, std::shared_ptr<wangle::ThreadPoolExecutor::Thread>)>::__call<void, , 0ul, 1ul>(std::tuple<>&&, std::_Index_tuple<0ul, 1ul>) (this=0xe62b40, __args=<unknown type in /data2/CppSrc/projects/cmake-build-debug/wangleserver, CU 0x1627bd1, DIE 0x167103f>) at /opt/rh/devtoolset-4/root/usr/include/c++/5.3.1/functional:1074
#21 0x000000000075aa4c in std::_Bind<std::_Mem_fn<void (wangle::ThreadPoolExecutor::*)(std::shared_ptr<wangle::ThreadPoolExecutor::Thread>)> (wangle::ThreadPoolExecutor*, std::shared_ptr<wangle::ThreadPoolExecutor::Thread>)>::operator()<, void>() (this=0xe62b40) at /opt/rh/devtoolset-4/root/usr/include/c++/5.3.1/functional:1133
#22 0x0000000000758ea4 in folly::detail::function::FunctionTraits<void ()>::callBig<std::_Bind<std::_Mem_fn<void (wangle::ThreadPoolExecutor::*)(std::shared_ptr<wangle::ThreadPoolExecutor::Thread>)> (wangle::ThreadPoolExecutor*, std::shared_ptr<wangle::ThreadPoolExecutor::Thread>)> >(folly::detail::function::Data&) (p=...) at /data2/CppSrc/projects/folly/folly/Function.h:296
#23 0x000000000050b55f in folly::detail::function::FunctionTraits<void ()>::operator()() (this=0xe62ba0) at /data2/CppSrc/projects/folly/folly/Function.h:305
#24 0x0000000000722e02 in std::_Bind_simple<folly::Function<void ()> ()>::_M_invoke<>(std::_Index_tuple<>) (this=0xe62ba0) at /opt/rh/devtoolset-4/root/usr/include/c++/5.3.1/functional:1531
#25 0x000000000072261b in std::_Bind_simple<folly::Function<void ()> ()>::operator()() (this=0xe62ba0) at /opt/rh/devtoolset-4/root/usr/include/c++/5.3.1/functional:1520
#26 0x000000000072216c in std::thread::_Impl<std::_Bind_simple<folly::Function<void ()> ()> >::_M_run() (this=0xe62b80) at /opt/rh/devtoolset-4/root/usr/include/c++/5.3.1/thread:115
#27 0x00000000008e5720 in execute_native_thread_routine ()
#28 0x00007ffff7bc8dc5 in start_thread () from /lib64/libpthread.so.0
#29 0x00007ffff667273d in clone () from /lib64/libc.so.6
