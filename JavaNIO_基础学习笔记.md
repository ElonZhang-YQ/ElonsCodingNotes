## Java NIO 基础学习笔记

1. `Buffer`、`Channel`和`Selector`

   1. `Buffer`

      1. 和传统IO不同的是，传统IO是面向`Stream`流，NIO是面向`Buffer`进行操作。

      2. 常用的`Buffer`类型是`ByteBuffer`，将数据转成字节进行处理。实质上就是一个byte数组（`byte[]`）

      3. 重要的三个参数：`capacity`、`position`和`limit`，还有`mark`

         - `capacity`：容量，表示缓冲区的大小，一旦声明就不可修改
         - `position`：位置，表示缓冲区正在操作数据的位置
         - `limit`：界限，缓冲区中可以操作数据的大小（limit后面的数据不能读写）
         - `mark`：标记当前position的位置，如果有读写切换的情况下，再次切换回来可以记录之前Position的位置，可以通过`reset()`恢复到mark的位置
         - 0 < mark <= position <= limit <= capacity

      4. Buffer分为堆内存（JVM堆）和堆外内存/直接内存（Native堆）

         可以通过ByteBuffer.allocate(size)和ByteBuffer.allocateDirect(size)来声明堆内存和直接内存

         ~~~java
         /** 声明堆内存
          *  实际上底层调用了 new HeapByteBuffer(capacity, capacity, null);
          *  HeapByteBuffer的实例化方法是package private，所以无法通过new关键字的方法实例化HeapByteBuffer对象
          */
         ByteBuffer byteBuffer = ByteBuffer.allocate(capacity);
         
         /** 声明堆外内存（直接内存）
          *  实际上底层调用了 new DirectByteBuffer(capacity);
          *  DirectByteBuffer和HeapByteBuffer一样，实例化方法都是package private，无法通过new关键字实例化DirectByteBuffer对象
          */
         ByteBuffer byteBuffer = ByteBuffer.allocateDirect(capacity);
         ~~~

      5. 其实并不是直接内存的效率高于堆内存，没有达到一定量级的情况下，使用DirectByteBuffer体现不出优势。

      6. DirectByteBuffer使用场景

         - Java程序与本地磁盘、Socket传输数据
         - 大文件对象，不会受到堆内存大小限制
         - 不需要频繁创建，生命周期较长，能够重复使用的情况

      7. Buffer切换到读模式`flip()`

      8. Buffer切换到写模式`clear()`或`compact()`

   2. `Channel`

      1. `Channel`本身并不存储数据，只负责数据的运输。必须要和`Buffer`一起使用。

      2. 常用的几个Channel

         - `FileChannel`，读写文件中的数据

         ~~~java
         FileInputStream fis = new FileInputStream(FilePath);
         FileChannel fisChannel = fis.getChannel();
         
         FileOutputStream fos = new FileOutputStream(FilePath);
         FileChannel fosChannel = fos.getChannel();
         ~~~

         - `ServerSocketChannel`，监听新进来的TCP连接，每个新进来的TCP连接都会创建一个SocketChannel

         ~~~java
         /**
          *	打开Selector
          *  并且获取ServerSocketChannel
          */
         Selector selector = Selector.open();
         ServerSocketChannel socketChannel = ServerSocketChannel.open();
         
         /**
          *	因为Selector是非阻塞的，需要声明ServerSocketChannel为非阻塞
          *	然后将channel注册到selector上面
          */
         socketChannel.configureBlocking(false);
         socketChannel.register(selector, SelectionKey.OP_ACCEPT);
         
         /**
          *	同时绑定地址和端口
          */
         ServerSocket socket = socketChannel.socket();
         socket.bind(new InetSocketAddress(HOST, PORT));
         ~~~

         - `SocketChannel`，通过TCP读写网络中的数据

         ~~~java
         // 打开Selector
         Selector selector = Selector.open();
         // 绑定地址和端口号
         SocketChannel socketChannel = SocketChannel.open(new InetSocketAddress(HOST, PORT));
         // 设置为非阻塞
         socketChannel.configureBlocking(false);
         // 注册channel到selector上面
         socketChannel.register(selector, SelectionKey.OP_READ);
         ~~~

         1. `DatagramSocketChannel`，通过UDP读写网络中的数据

   3. `Selector`

      1. NIO多路复用的核心
      2. 用于监控`SelectableChannel`的IO状况
      3. 在一个线程中管理多个Channels
      4. channel向selector注册的4种事件类型
         - `SelectionKey.OP_READ`
           - 读取操作，值为1
         - `SelectionKey.OP_WRITE`
           - 写入操作，值为4
         - `SelectionKey.OP_CONNECT`
           - Socket连接操作，值为8
         - `SelectionKey.OP_ACCEPT`
           - Socket接受操作，值为16
         - 若注册时不止监听一个事件，可以使用“| 位或”操作符连接。
      5. SelectionKey的常用操作
      6. Selector的常用方法

   4. NIO内存映射

      1. 上面在将ByteBuffer的时候，讲到ByteBuffer可以有堆内存形式和直接内存的形式，内存映射就是直接内存的形式

      2. 有两种方式：

         - `ByteBuffer.allocateDirect(capacity);`

         - 通过`FileChannel`的`map()`获取`MappedByteBuffer`

           ~~~java
           /**
            *	打开文件输入输出流
            */
           FileChannel inChannel = FileChannel.open(Paths.get(currentFilePath), StandardOpenOption.READ);
           FileChannel outChannel = FileChannel.open(Paths.get(newFilePath), StandardOpenOption.WRITE, StandardOpenOption.READ, StandardOpenOption.CREATE);
           
           /**
            *	通过文件的输入输出流，获取MappedByteBuffer
            */
           MappedByteBuffer inMapped = inChannel.map(FileChannel.MapMode.READ_ONLY, 0, 2048);
           MappedByteBuffer outMapped = outChannel.map(FileChannel.MapMode.READ_WRITE, 0, 2048);
           
           /**
            *	通过字节数组，将inputChannel的数据读取到outputChannel中
            */
           byte[] dst = new byte[inMapped.limit()];
           inMapped.get(dst);
           outMapped.put(dst);
           ~~~

         - 两种方式对于同一文档，相同capacity的情况下，读取时间没差。

   5. NIO零拷贝

      1. 正常的IO操作中，需要进行多次的用户态和内核态的read()和write()操作。导致了额外的CPU开销，降低了IO性能。

      2. 零拷贝技术可以直接在内核态中进行数据的处理，避免了数据的复制操作。

      3. 通过`transferTo`方法可以将数据从源通道传输到目标通道，传输过程中间不需要在用户空间和内核空间复制数据。

         ~~~java
         FileChannel inputChannel = new FileInputStream(currentFilePath).getChannel();
         FileChannel outputChannel = new FileOutputStream(newFilePath).getChannel();
         inputChannel.transferTo(0, 2048, outputChannel);
         
         inputChannel.close();
         outputChannel.close();
         ~~~

         

