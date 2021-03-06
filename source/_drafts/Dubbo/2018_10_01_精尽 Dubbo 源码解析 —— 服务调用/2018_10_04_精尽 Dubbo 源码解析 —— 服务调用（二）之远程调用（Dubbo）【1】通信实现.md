title: 精尽 Dubbo 源码分析 —— 服务调用（二）之远程调用（Dubbo）【1】通信实现
date: 2018-10-04
tags:
categories: Dubbo
permalink: Dubbo/rpc-dubbo-1-remoting

-------

摘要: 原创出处 http://www.iocoder.cn/Dubbo/rpc-dubbo-1-remoting/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Dubbo/rpc-dubbo-1-remoting/)
- [2. Server](http://www.iocoder.cn/Dubbo/rpc-dubbo-1-remoting/)
- [3. Client](http://www.iocoder.cn/Dubbo/rpc-dubbo-1-remoting/)
- [4. ExchangeHandler](http://www.iocoder.cn/Dubbo/rpc-dubbo-1-remoting/)
- [5. Codec](http://www.iocoder.cn/Dubbo/rpc-dubbo-1-remoting/)
  - [5.1 DubboCountCodec](http://www.iocoder.cn/Dubbo/rpc-dubbo-1-remoting/)
  - [5.2 DubboCodec](http://www.iocoder.cn/Dubbo/rpc-dubbo-1-remoting/)
  - [5.3 DecodeableRpcInvocation](http://www.iocoder.cn/Dubbo/rpc-dubbo-1-remoting/)
  - [5.4 DecodeableRpcResult](http://www.iocoder.cn/Dubbo/rpc-dubbo-1-remoting/)
- [666. 彩蛋](http://www.iocoder.cn/Dubbo/rpc-dubbo-1-remoting/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

# 1. 概述

从本文开始，我们开始分享 `dubbo://` 协议的远程调用，主要分成**四个部分**：

1. 通信实现
2. 同步调用
3. 异步调用
4. 参数回调

本文分享 **通信实现** 部分。

😈 [《精尽 Dubbo 源码解析 —— NIO 服务器》](#) 系列，是本文的**前置文章**，所以胖友需要先读完这个系列。哈哈哈，当然，也可以凑合看看先。

本文涉及类图如下：

![类图](http://www.iocoder.cn/images/Dubbo/2018_10_04/01_01.png)

# 2. Server

在 [《精尽 Dubbo 源码分析 —— 服务引用（二）之远程暴露（Dubbo）》](http://www.iocoder.cn/Dubbo/reference-export-dubbo/?self) 中，我们看到使用的 Server 实现类是 **HeaderExchangeServer** 。

# 3. Client

在 [《精尽 Dubbo 源码分析 —— 服务引用（二）之远程引用（Dubbo）》](http://www.iocoder.cn/Dubbo/reference-refer-dubbo/?self) 中，我们看到使用的 Client 实现类是 **ReferenceCountExchangeClient** 和 **LazyConnectExchangeClient** 。

# 4. ExchangeHandler

在 DubboProtocol 中，实现了 ExchangeHandler ，代码如下：

```Java
private ExchangeHandler requestHandler = new ExchangeHandlerAdapter() {

    @Override
    public Object reply(ExchangeChannel channel, Object message) throws RemotingException {
        // ... 省略具体实现
    }

    @Override
    public void received(Channel channel, Object message) throws RemotingException {
        // ... 省略具体实现
    }

    @Override
    public void connected(Channel channel) throws RemotingException {
        this.invoke(channel, Constants.ON_CONNECT_KEY);
    }

    @Override
    public void disconnected(Channel channel) throws RemotingException {
        // ... 省略具体实现
    }

    private void invoke(Channel channel, String methodKey) {
        // ... 省略具体实现
    }

};
```

这个处理器，负责将请求，**转发到对应的 Invoker 对象**，执行逻辑，返回结果。    
当然，本文不细分享，放在 **同步调用** 一文详细解析。

# 5. Codec

在 [ExchangeCodec](https://github.com/apache/incubator-dubbo/blob/master/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/exchange/codec/ExchangeCodec.java) 中，我们看到对 Request 和 Response 的**通用**解析。但是它是**不满足**在 `dubbo://` 协议中，对 [RpcInvocation]() 和 [RpcResult]() 作为 **内容体( Body )** 的编解码的需要的。

另外，在 `dubbo://` 协议中，支持 [参数回调](https://dubbo.gitbooks.io/dubbo-user-book/demos/callback-parameter.html) 的特性，也是需要在编解码做一些**特殊逻辑**。

下面，让我们来一起瞅瞅代码实现吧。

## 5.1 DubboCountCodec

[`com.alibaba.dubbo.rpc.protocol.dubbo.DubboCountCodec`](https://github.com/YunaiV/dubbo/blob/master/dubbo-rpc/dubbo-rpc-default/src/main/java/com/alibaba/dubbo/rpc/protocol/dubbo/DubboCountCodec.java) ，实现 Codec2 接口，支持**多消息**的编解码器。

### 5.1.1 构造方法

```Java
/**
 * 编解码器
 */
private DubboCodec codec = new DubboCodec();
```

* 在 Dubbo Client 和 Server 创建的过程，我们看到设置了编解码器为 `"dubbo"` ，从而通过 Dubbo SPI 机制，加载到 DubboCountCodec 。相关内容如下：

    ```Java
    // DubboProtocol#createServer(...)
    url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
    
    // DubboProtocol#initClient(...)
    url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
    
    // META-INF/dubbo/internal/com.alibaba.dubbo.remoting.Codec2
    dubbo=com.alibaba.dubbo.rpc.protocol.dubbo.DubboCountCodec
    ```

* 实际编解码的逻辑，使用 DubboCodec ，即 `codec` 属性。

### 5.1.2 编码

```Java
@Override
public void encode(Channel channel, ChannelBuffer buffer, Object msg) throws IOException {
    codec.encode(channel, buffer, msg);
}
```

### 5.1.3 解码

```Java
  1: @Override
  2: public Object decode(Channel channel, ChannelBuffer buffer) throws IOException {
  3:     // 记录当前读位置
  4:     int save = buffer.readerIndex();
  5:     // 创建 MultiMessage 对象
  6:     MultiMessage result = MultiMessage.create();
  7:     do {
  8:         // 解码
  9:         Object obj = codec.decode(channel, buffer);
 10:         // 输入不够，重置读进度
 11:         if (Codec2.DecodeResult.NEED_MORE_INPUT == obj) {
 12:             buffer.readerIndex(save);
 13:             break;
 14:         // 解析到消息
 15:         } else {
 16:             // 添加结果消息
 17:             result.addMessage(obj);
 18:             // 记录消息长度到隐式参数集合，用于 MonitorFilter 监控
 19:             logMessageLength(obj, buffer.readerIndex() - save);
 20:             // 记录当前读位置
 21:             save = buffer.readerIndex();
 22:         }
 23:     } while (true);
 24:     // 需要更多的输入
 25:     if (result.isEmpty()) {
 26:         return Codec2.DecodeResult.NEED_MORE_INPUT;
 27:     }
 28:     // 返回解析到的消息
 29:     if (result.size() == 1) {
 30:         return result.get(0);
 31:     }
 32:     return result;
 33: }
```

* 包含两块逻辑：1）多消息解析的支持。2）记录每条消息的长度，用于 MonitorFilter 监控。
* 第 4 行：记录当前读位置，用于下面计算每条消息的长度。
* 第 6 行：创建 MultiMessage 对象。MultiMessageHandler 支持对它的处理分发。
* 第 7 至 23 行：**循环**解析消息，直到结束。
* 第 9 行：调用 `DubboCodec#decode(channel, buffer)` 方法，解码。
* 第 11 至 13 行：字节数组不够，重置读进度，结束解析。
* 第 15 至 22 行：解析到消息，添加到 `result` 。
    * 第 19 行：调用 `#logMessageLength(obj, length)`  方法，记录消息长度到**隐式参数集合**，用于 MonitorFilter 监控。代码如下：

        ```Java
        private void logMessageLength(Object result, int bytes) {
            if (bytes <= 0) {
                return;
            }
            if (result instanceof Request) {
                try {
                    ((RpcInvocation) ((Request) result).getData()).setAttachment(Constants.INPUT_KEY, String.valueOf(bytes)); // 请求
                } catch (Throwable e) {
                    /* ignore */
                }
            } else if (result instanceof Response) {
                try {
                    ((RpcResult) ((Response) result).getResult()).setAttachment(Constants.OUTPUT_KEY, String.valueOf(bytes)); // 响应
                } catch (Throwable e) {
                    /* ignore */
                }
            }
        }
        ```
        * x
    * 第 21 行：记录当前读位置，用于计算**下一条**消息的长度。
* 第 24 至 27 行：需要更多的输入。
* 第 28 至 32 行：返回结果。

## 5.2 DubboCodec

[`com.alibaba.dubbo.rpc.protocol.dubbo.DubboCodec`](https://github.com/YunaiV/dubbo/blob/master/dubbo-rpc/dubbo-rpc-default/src/main/java/com/alibaba/dubbo/rpc/protocol/dubbo/DubboCountCodec.java) ，实现 Codec2 接口，继承 ExchangeCodec 类，**Dubbo 编解码器**实现类。

### 5.2.1 构造方法

```Java
/**
 * 协议名
 */
public static final String NAME = "dubbo";
/**
 * 协议版本
 */
public static final String DUBBO_VERSION = Version.getVersion(DubboCodec.class, Version.getVersion());

/**
 * 响应 - 异常
 */
public static final byte RESPONSE_WITH_EXCEPTION = 0;
/**
 * 响应 - 正常（空返回）
 */
public static final byte RESPONSE_VALUE = 1;
/**
 * 响应 - 正常（有返回）
 */
public static final byte RESPONSE_NULL_VALUE = 2;

/**
 * 方法参数 - 空（参数）
 */
public static final Object[] EMPTY_OBJECT_ARRAY = new Object[0];
/**
 * 方法参数 - 空（类型）
 */
public static final Class<?>[] EMPTY_CLASS_ARRAY = new Class<?>[0];
```

### 5.2.2 编码内容体

#### 5.2.2.1 请求

```Java
  1: @Override
  2: protected void encodeRequestData(Channel channel, ObjectOutput out, Object data) throws IOException {
  3:     RpcInvocation inv = (RpcInvocation) data;
  4: 
  5:     // 写入 `dubbo` `path` `version`
  6:     out.writeUTF(inv.getAttachment(Constants.DUBBO_VERSION_KEY, DUBBO_VERSION));
  7:     out.writeUTF(inv.getAttachment(Constants.PATH_KEY));
  8:     out.writeUTF(inv.getAttachment(Constants.VERSION_KEY));
  9: 
 10:     // 写入方法、方法签名、方法参数集合
 11:     out.writeUTF(inv.getMethodName());
 12:     out.writeUTF(ReflectUtils.getDesc(inv.getParameterTypes()));
 13:     Object[] args = inv.getArguments();
 14:     if (args != null) {
 15:         for (int i = 0; i < args.length; i++) {
 16:             out.writeObject(CallbackServiceCodec.encodeInvocationArgument(channel, inv, i));
 17:         }
 18:     }
 19: 
 20:     // 写入隐式传参集合
 21:     out.writeObject(inv.getAttachments());
 22: }
```

* 🙂 胖友看下代码注释。
* 编码 RpcInvocation 对象，写入需要编码的字段。
* 对应的解码，在 DecodeableRpcInvocation 中。
* 第 16 行：调用 `CallbackServiceCodec#encodeInvocationArgument(...)` 方法，编码参数。主要用于 [参数回调](https://dubbo.gitbooks.io/dubbo-user-book/demos/callback-parameter.html) 功能，后面的文章，详细解析。

#### 5.2.2.2 响应

```Java
  1: @Override
  2: protected void encodeResponseData(Channel channel, ObjectOutput out, Object data) throws IOException {
  3:     Result result = (Result) data;
  4: 
  5:     Throwable th = result.getException();
  6:     // 正常
  7:     if (th == null) {
  8:         Object ret = result.getValue();
  9:         // 空返回
 10:         if (ret == null) {
 11:             out.writeByte(RESPONSE_NULL_VALUE);
 12:         // 有返回
 13:         } else {
 14:             out.writeByte(RESPONSE_VALUE);
 15:             out.writeObject(ret);
 16:         }
 17:     // 异常
 18:     } else {
 19:         out.writeByte(RESPONSE_WITH_EXCEPTION);
 20:         out.writeObject(th);
 21:     }
 22: }
```

* 🙂 胖友看下代码注释。
* 编码 Result 对象，写入需要编码的字段。
* 对应的解码，在 DecodeableRpcResult 中。

### 5.2.3 解码内容体

```Java
  1: @Override
  2: protected Object decodeBody(Channel channel, InputStream is, byte[] header) throws IOException {
  3:     byte flag = header[2];
  4:     // 获得 Serialization 对象
  5:     byte proto = (byte) (flag & SERIALIZATION_MASK);
  6:     Serialization s = CodecSupport.getSerialization(channel.getUrl(), proto);
  7:     // 获得请求||响应编号
  8:     // get request id.
  9:     long id = Bytes.bytes2long(header, 4);
 10:     // 解析响应
 11:     if ((flag & FLAG_REQUEST) == 0) {
 12:         // decode response.
 13:         Response res = new Response(id);
 14:         // ... 省略代码
 15:         return res;
 16:     // 解析请求
 17:     } else {
 18:         // decode request.
 19:         Request req = new Request(id);
 20:         // ... 省略代码
 21:         return req;
 22:     }
 23: }
```

* 第 4 至 6 行：调用 `CodeSupport#getSerialization(url, proto)` 方法，获得 Serialization 对象，用于下面反序列化内容体的每个字段。
* 第 9 行：获得请求或响应的编号。
* 第 10 至 15 行：解析响应( Response )。
* 第 16 至 22 行：解析请求( Request )。

#### 5.2.3.1 请求

```Java
  1: // decode response.
  2: Response res = new Response(id);
  3: // 若是心跳事件，进行设置
  4: if ((flag & FLAG_EVENT) != 0) {
  5:     res.setEvent(Response.HEARTBEAT_EVENT);
  6: }
  7: // 设置状态
  8: // get status.
  9: byte status = header[3];
 10: res.setStatus(status);
 11: // 正常响应状态
 12: if (status == Response.OK) {
 13:     try {
 14:         Object data;
 15:         // 解码心跳事件
 16:         if (res.isHeartbeat()) {
 17:             data = decodeHeartbeatData(channel, deserialize(s, channel.getUrl(), is));
 18:         // 解码其它事件
 19:         } else if (res.isEvent()) {
 20:             data = decodeEventData(channel, deserialize(s, channel.getUrl(), is));
 21:         // 解码普通响应
 22:         } else {
 23:             DecodeableRpcResult result;
 24:             // 在通信框架（例如，Netty）的 IO 线程，解码
 25:             if (channel.getUrl().getParameter(Constants.DECODE_IN_IO_THREAD_KEY, Constants.DEFAULT_DECODE_IN_IO_THREAD)) {
 26:                 result = new DecodeableRpcResult(channel, res, is, (Invocation) getRequestData(id), proto);
 27:                 result.decode();
 28:             // 在 Dubbo ThreadPool 线程，解码，使用 DecodeHandler
 29:             } else {
 30:                 result = new DecodeableRpcResult(channel, res, new UnsafeByteArrayInputStream(readMessageData(is)), (Invocation) getRequestData(id), proto);
 31:             }
 32:             data = result;
 33:         }
 34:         // 设置结果
 35:         res.setResult(data);
 36:     } catch (Throwable t) {
 37:         if (log.isWarnEnabled()) {
 38:             log.warn("Decode response failed: " + t.getMessage(), t);
 39:         }
 40:         res.setStatus(Response.CLIENT_ERROR);
 41:         res.setErrorMessage(StringUtils.toString(t));
 42:     }
 43: // 异常响应状态
 44: } else {
 45:     res.setErrorMessage(deserialize(s, channel.getUrl(), is).readUTF());
 46: }
 47: return res;
```

* 🙂 胖友看下代码注释。我们重点讲下可能**比较绕**的地方。
* 第 21 至 33 行：解码普通响应。我们可以看到代码分成【第 25 至 27 行】【第 28 至 31 行】**两段**。
    * 相同点，使用  **DecodeableRpcResult** 解码。前者，比较好理解，【第 27 行】已经调用；后者，在 DecodeHandler 中，才最终调用 `DecodeableRpcResult#decode()` 方法。
    * 差异点，使用**哪个线程**解码。前者，还是比较好理解，当前线程，即通信框架（例如，Netty）的 IO 线程。后者，Dubbo ThreadPool 线程中。
    * `decode.in.io` 配置项，目前在 Dubbo 文档中，并未说明，应该是**性能调优**，具体笔者还没测试过。嘿嘿。

#### 5.2.3.2 响应

```Java
  1: // decode request.
  2: Request req = new Request(id);
  3: req.setVersion("2.0.0");
  4: // 是否需要响应
  5: req.setTwoWay((flag & FLAG_TWOWAY) != 0);
  6: // 若是心跳事件，进行设置
  7: if ((flag & FLAG_EVENT) != 0) {
  8:     req.setEvent(Request.HEARTBEAT_EVENT);
  9: }
 10: try {
 11:     Object data;
 12:     // 解码心跳事件
 13:     if (req.isHeartbeat()) {
 14:         data = decodeHeartbeatData(channel, deserialize(s, channel.getUrl(), is));
 15:     // 解码其它事件
 16:     } else if (req.isEvent()) {
 17:         data = decodeEventData(channel, deserialize(s, channel.getUrl(), is));
 18:     // 解码普通请求
 19:     } else {
 20:         // 在通信框架（例如，Netty）的 IO 线程，解码
 21:         DecodeableRpcInvocation inv;
 22:         if (channel.getUrl().getParameter(Constants.DECODE_IN_IO_THREAD_KEY, Constants.DEFAULT_DECODE_IN_IO_THREAD)) {
 23:             inv = new DecodeableRpcInvocation(channel, req, is, proto);
 24:             inv.decode();
 25:         // 在 Dubbo ThreadPool 线程，解码，使用 DecodeHandler
 26:         } else {
 27:             inv = new DecodeableRpcInvocation(channel, req, new UnsafeByteArrayInputStream(readMessageData(is)), proto);
 28:         }
 29:         data = inv;
 30:     }
 31:     req.setData(data);
 32: } catch (Throwable t) {
 33:     if (log.isWarnEnabled()) {
 34:         log.warn("Decode request failed: " + t.getMessage(), t);
 35:     }
 36:     // bad request
 37:     req.setBroken(true);
 38:     req.setData(t);
 39: }
 40: return req;
```

* 和 [「5.2.3.1 请求」](#) **类似**，差异点在使用 **DecodeableRpcInvocation** 。
* 🙂 胖友看下代码注释。

## 5.3 DecodeableRpcInvocation

[`com.alibaba.dubbo.rpc.protocol.dubbo.DecodeableRpcInvocation`](https://github.com/YunaiV/dubbo/blob/master/dubbo-rpc/dubbo-rpc-default/src/main/java/com/alibaba/dubbo/rpc/protocol/dubbo/DecodeableRpcInvocation.java) ，实现 Codec 和 Decodeable 接口，继承 RpcInvocation 类，**可解码**的 RpcInvocation 实现类。

当服务消费者，调用服务提供者，前者编码的 RpcInvocation 对象，后者解码成 DecodeableRpcInvocation 对象。

从目前的代码实现来看，Codec 接口，可不实现。

### 5.3.1 构造方法

```Java
/**
 * 通道
 */
private Channel channel;
/**
 * Serialization 类型编号
 */
private byte serializationType;
/**
 * 输入流
 */
private InputStream inputStream;
/**
 * 请求
 */
private Request request;
/**
 * 是否已经解码完成
 */
private volatile boolean hasDecoded;
```

### 5.3.2 解码

```Java
@Override
public void decode() {
    if (!hasDecoded && channel != null && inputStream != null) {
        try {
            decode(channel, inputStream);
        } catch (Throwable e) {
            if (log.isWarnEnabled()) {
                log.warn("Decode rpc invocation failed: " + e.getMessage(), e);
            }
            request.setBroken(true);
            request.setData(e);
        } finally {
            hasDecoded = true;
        }
    }
}

@Override
public Object decode(Channel channel, InputStream input) throws IOException {
    ObjectInput in = CodecSupport.getSerialization(channel.getUrl(), serializationType).deserialize(channel.getUrl(), input);

    // 解码 `dubbo` `path` `version`
    setAttachment(Constants.DUBBO_VERSION_KEY, in.readUTF());
    setAttachment(Constants.PATH_KEY, in.readUTF());
    setAttachment(Constants.VERSION_KEY, in.readUTF());

    // 解码方法、方法签名、方法参数集合
    setMethodName(in.readUTF());
    try {
        Object[] args;
        Class<?>[] pts;
        String desc = in.readUTF();
        if (desc.length() == 0) {
            pts = DubboCodec.EMPTY_CLASS_ARRAY;
            args = DubboCodec.EMPTY_OBJECT_ARRAY;
        } else {
            pts = ReflectUtils.desc2classArray(desc);
            args = new Object[pts.length];
            for (int i = 0; i < args.length; i++) {
                try {
                    args[i] = in.readObject(pts[i]);
                } catch (Exception e) {
                    if (log.isWarnEnabled()) {
                        log.warn("Decode argument failed: " + e.getMessage(), e);
                    }
                }
            }
        }
        setParameterTypes(pts);

        // 解码隐式传参集合
        Map<String, String> map = (Map<String, String>) in.readObject(Map.class);
        if (map != null && map.size() > 0) {
            Map<String, String> attachment = getAttachments();
            if (attachment == null) {
                attachment = new HashMap<String, String>();
            }
            attachment.putAll(map);
            setAttachments(attachment);
        }

        // 进一步解码方法参数，主要为了参数返回
        // decode argument ,may be callback
        for (int i = 0; i < args.length; i++) {
            args[i] = CallbackServiceCodec.decodeInvocationArgument(channel, this, pts, i, args[i]);
        }
        setArguments(args);
    } catch (ClassNotFoundException e) {
        throw new IOException(StringUtils.toString("Read invocation data failed.", e));
    } finally {
        if (in instanceof Cleanable) {
            ((Cleanable) in).cleanup();
        }
    }
    return this;
}
```

* 🙂 胖友看下代码注释。

## 5.4 DecodeableRpcResult

> 和 DecodeableRpcInvocation 一致。

[`com.alibaba.dubbo.rpc.protocol.dubbo.DecodeableRpcResult`](https://github.com/YunaiV/dubbo/blob/master/dubbo-rpc/dubbo-rpc-default/src/main/java/com/alibaba/dubbo/rpc/protocol/dubbo/DecodeableRpcResult.java) ，实现 Codec 和 Decodeable 接口，继承 RpcResult 类，**可解码**的 RpcResult 实现类。

当服务提供者者，返回服务消费者调用结果，前者编码的 RpcResult 对象，后者解码成 DecodeableRpcResult 对象。

从目前的代码实现来看，Codec 接口，可不实现。

### 5.4.1 构造方法

```Java
/**
 * 通道
 */
private Channel channel;
/**
 * Serialization 类型编号
 */
private byte serializationType;
/**
 * 输入流
 */
private InputStream inputStream;
/**
 * 请求
 */
private Response response;
/**
 * Invocation 对象
 */
private Invocation invocation;
/**
 * 是否已经解码完成
 */
private volatile boolean hasDecoded;
```

### 5.4.2 解码

```Java
@Override
public void decode() {
    if (!hasDecoded && channel != null && inputStream != null) {
        try {
            decode(channel, inputStream);
        } catch (Throwable e) {
            if (log.isWarnEnabled()) {
                log.warn("Decode rpc result failed: " + e.getMessage(), e);
            }
            response.setStatus(Response.CLIENT_ERROR);
            response.setErrorMessage(StringUtils.toString(e));
        } finally {
            hasDecoded = true;
        }
    }
}

@Override
public Object decode(Channel channel, InputStream input) throws IOException {
    ObjectInput in = CodecSupport.getSerialization(channel.getUrl(), serializationType).deserialize(channel.getUrl(), input);

    // 读取标记位
    byte flag = in.readByte();
    switch (flag) {
        case DubboCodec.RESPONSE_NULL_VALUE: // 无返回值
            break;
        case DubboCodec.RESPONSE_VALUE: // 有返回值
            try {
                Type[] returnType = RpcUtils.getReturnTypes(invocation);
                setValue(returnType == null || returnType.length == 0 ? in.readObject() :
                        (returnType.length == 1 ? in.readObject((Class<?>) returnType[0])
                                // 返回结果:Type[]{method.getReturnType(), method.getGenericReturnType()}
                                : in.readObject((Class<?>) returnType[0], returnType[1])));
            } catch (ClassNotFoundException e) {
                throw new IOException(StringUtils.toString("Read response data failed.", e));
            }
            break;
        case DubboCodec.RESPONSE_WITH_EXCEPTION: // 异常
            try {
                Object obj = in.readObject();
                if (!(obj instanceof Throwable)) {
                    throw new IOException("Response data error, expect Throwable, but get " + obj);
                }
                setException((Throwable) obj);
            } catch (ClassNotFoundException e) {
                throw new IOException(StringUtils.toString("Read response data failed.", e));
            }
            break;
        default:
            throw new IOException("Unknown result flag, expect '0' '1' '2', get " + flag);
    }
    if (in instanceof Cleanable) {
        ((Cleanable) in).cleanup();
    }
    return this;
}
```

* 🙂 胖友看下代码注释。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

清明节，扫代码第一波。

