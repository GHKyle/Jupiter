#### 传输层
- 传输层目前只提供了Netty4.1.x的实现, 理论上完全可以替换到其他Nio框架, 其他模块对Netty没有任何依赖

###### Protocol

*****************************************************************************
                                            Protocol
    ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐
         2   │   1   │    1   │     8     │      4      │
    ├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┤
             │       │        │           │             │
    │  MAGIC   Sign    Status   Invoke Id   Body Length                Body Content             │
             │       │        │           │             │
    └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘


    ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐
        * 消息头16个字节定长
         * = 2 // magic = (short) 0xbabe 
        * + 1 // 消息标志位, 低地址4位用来表示消息类型request/response/heartbeat等, 高地址4位用来表示序列化类型
        * + 1 // 状态位, 设置请求响应状态
        * + 8 // 消息 id, long 类型 
        * + 4 // 消息体 body 长度, int 类型
    └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘

###### Acceptor

*****************************************************************************
                  I/O Request                       I/O Response
                       │                                 △
                                                         │
                       │
       ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┼ ─ ─ ─ ─ ─ ─ ─ ─
       │               │                                                  │
                                                         │
       │  ┌ ─ ─ ─ ─ ─ ─▽─ ─ ─ ─ ─ ─ ┐       ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐   │
           IdleStateChecker#inBound          IdleStateChecker#outBound
       │  └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘       └ ─ ─ ─ ─ ─ ─△─ ─ ─ ─ ─ ─ ┘   │
                       │                                 │
       │                                                                  │
                       │                                 │
       │  ┌ ─ ─ ─ ─ ─ ─▽─ ─ ─ ─ ─ ─ ┐                                     │
           AcceptorIdleStateTrigger                      │
       │  └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘                                     │
                       │                                 │
       │                                                                  │
                       │                                 │
       │  ┌ ─ ─ ─ ─ ─ ─▽─ ─ ─ ─ ─ ─ ┐       ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐   │
                ProtocolDecoder                   ProtocolEncoder
       │  └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘       └ ─ ─ ─ ─ ─ ─△─ ─ ─ ─ ─ ─ ┘   │
                       │                                 │
       │                                                                  │
                       │                                 │
       │  ┌ ─ ─ ─ ─ ─ ─▽─ ─ ─ ─ ─ ─ ┐                                     │
                AcceptorHandler                          │
       │  └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘                                     │
                       │                                 │
       │                    ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐                     │
                       ▽                                 │
       │               ─ ─ ▷│       Processor       ├ ─ ─▷                │

       │                    └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘                     │
       ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─


###### Connector

*****************************************************************************
                            ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐

                       ─ ─ ─│        Server         │─ ─▷
                       │                                 │
                            └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘
                       │                                 ▽
                                                    I/O Response
                       │                                 │

                       │                                 │
       ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
       │               │                                 │                │

       │               │                                 │                │
         ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐      ┌ ─ ─ ─ ─ ─ ─▽─ ─ ─ ─ ─ ─ ─
       │  ConnectionWatchdog#outbound        ConnectionWatchdog#inbound│  │
         └ ─ ─ ─ ─ ─ ─ △ ─ ─ ─ ─ ─ ─ ┘      └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
       │                                                 │                │
                       │
       │  ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐       ┌ ─ ─ ─ ─ ─ ─▽─ ─ ─ ─ ─ ─ ┐   │
           IdleStateChecker#outBound         IdleStateChecker#inBound
       │  └ ─ ─ ─ ─ ─ ─△─ ─ ─ ─ ─ ─ ┘       └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘   │
                                                         │
       │               │                                                  │
                                            ┌ ─ ─ ─ ─ ─ ─▽─ ─ ─ ─ ─ ─ ┐
       │               │                     ConnectorIdleStateTrigger    │
                                            └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘
       │               │                                 │                │

       │               │                    ┌ ─ ─ ─ ─ ─ ─▽─ ─ ─ ─ ─ ─ ┐   │
                                                  ProtocolDecoder
       │               │                    └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘   │
                                                         │
       │               │                                                  │
                                            ┌ ─ ─ ─ ─ ─ ─▽─ ─ ─ ─ ─ ─ ┐
       │               │                         ConnectorHandler         │
          ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐       └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘
       │        ProtocolEncoder                          │                │
          └ ─ ─ ─ ─ ─ ─△─ ─ ─ ─ ─ ─ ┘
       │                                                 │                │
       ─ ─ ─ ─ ─ ─ ─ ─ ┼ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
                                             ┌ ─ ─ ─ ─ ─ ▽ ─ ─ ─ ─ ─ ┐
                       │
                                             │       Processor       │
                       │
                  I/O Request                └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘