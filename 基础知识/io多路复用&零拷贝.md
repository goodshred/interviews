## 零拷贝
>结论

mmap支持修改数据，sendfile不支持修改，一般写用mmap，java零拷贝支持sendfile和mmap，kafka用的是java的sendfile零拷贝，还有一种linux2.6.7版本后支持的splice零拷贝技术但是java不支持

>场景

sendfile调用有一个缺点，那就是无法在sendfile调用过程中修改数据，因此sendfile()只是适用于应用程序地址空间不需要对所访问数据进行处理的和修改情况，常见的就是文件传输，或者MQ消费消息的获取
如果只是简单地从一个文件描述符读取数据并将其写入另一个文件描述符，则可以使
用splice实现零拷贝传输，以获得更好的性能。如果需要更复杂的操作，如在读取数据之前
需要对数据进行处理或需要将数据写入文件，可能需要使用sendfile 或其他类似的技术。


>概念

所谓的零拷贝是指将数据在内核空间直接从磁盘文件复制到网卡中，而不需要经由用户态的应用程序之手。 这样既可以提高数据读取的性能，也能减少核心态和用户态之间的上下文切换，提高数据传输效率。

>分析

总结一下：Kafka用到的零拷贝技术，主要是减少了核心态和用户态之间的两次数据拷贝过程，使得数据可以不用经过用户态直接通过网卡发送到接收方，同时通过DMA技术，可以使CPU得到解放，这样实现了数据的高性能传输。

Java中的零拷贝是依靠java.nio.channels.FileChannel中的transferTo(long position, long count, WritableByteChannel target)方法来实现的。transferTo方法的底层实现是基于操作系统的sendfile这个system call来实现的，无需将数据拷贝到用户态，sendfile负责把数据从某个fd（file descriptor）传输到另一个fd。这样就完成了零拷贝的过程。零拷贝的示例代码如下所示：


```java
import java.io.File;
import java.io.RandomAccessFile;
import java.net.InetSocketAddress;
import java.nio.channels.FileChannel;
import java.nio.channels.SocketChannel;

public class ZeroCopy {
    public static void main(String[] args) throws Exception {
        File file  = new File("xxxxxx.log");

        RandomAccessFile raf = new RandomAccessFile(file, "rw");

        FileChannel channel = raf.getChannel();

        //Opens a socket channel and connects it to a remote address.
        SocketChannel socketChannel = SocketChannel.open(
                new InetSocketAddress("192.168.2.222", 9091)
        );

        //Transfers bytes from this channel's file to the given writable byte channel.
        channel.transferTo(0,channel.size(), socketChannel);
    }
}
```

## bio-nio-aio
场景说明同步异步，阻塞非阻塞：我打电话咨询书店老板有没有红楼梦这本书，书店老板需要找一找再给我答复

同步异步：发生在我和书店老板之间：

    同步：我打电话给老板，我每隔1秒就问老板查到这本书了吗，我不能挂电话，否则我和老板就中断联系了，老板不会打电话给我（老板告诉我有，我要亲自去店里买书再带回家）
    
    异步：我打电话给老板，问有这本书吗，老板说找找一会打给我，那我可以马上挂电话，然后干别的，老板过会会打给我（我把我的电话号码给老板，以及我的地址给老板，我想干的事情也告诉老板：老板找到书后，电话通知我，然后他自己把书按地址给我送上门）

阻塞非阻塞：只发生在我：

    阻塞：我要不间断问老板，不能挂电话，也不能做的事情，只能等着老板查了之后我轮询到结果才能去做别的事情，否则只能等
    
    非阻塞：我把电话打了，然后我就可以挂电话，去接着做别的事情

```注脚
内核异步io事件：老板在图书管理系统上输入红楼梦，系统（网卡）查到后会短信通知老板

用户态（用户线程：买书的我）--内核态（内核线程：老板：cpu）--外设（磁盘，网卡：驱动程序）
CPU控制外设，既需要硬件，也需要软件。硬件就是上面所讲的控制器、适配卡等等。软件主要就是各种外设的驱动程序和各种中断处理程序，驱动程序和中断程序统称为：设备处理程序

内核态和用户态可以理解成内核态线程（可以操作io外设，进行内存分配等），用户态线程（无外设的操作权限，只能在指定内存区域做事情）
```


bio 用户态需要去轮询内核（用户程序得不断去询问内核有连接上来了吗，可读了吗，可写了吗）

nio 内核提供了回调机制，轮询的事情内核去做了（内核会通知用户程序有连接上来了，可读了，可写了，并执行相关的回调操作）

aio window系统支持，但linux不支持，netty实现了，但linux系统底层的实现还是nio（epoll模式）

>bio阻塞的地方：等待网络连接，等待网络流的读，等待网络流的写

>nio快的原因：操作系统提供了这样一个接口：使得之前的多次系统调用变成一次系统调用批量处理


### select和epoll执行过程及原理
>select

1. select 调用需要传入 fd 数组，需要拷贝一份到内核，高并发场景下这样的拷贝消耗的资源是惊人的。（可优化为不复制）
2. select 在内核层仍然是通过遍历的方式检查文件描述符的就绪状态，是个同步过程，只不过无系统调用切换上下文的开销。（内核层可优化为异步事件通知）
3. select 仅仅返回可读文件描述符的个数，具体哪个可读还是要用户自己遍历。（可优化为只返回给用户就绪的文件描述符，无需用户做无效的遍历）

select 实现多路复用的方式是，将已连接的 Socket 都放到一个文件描述符集合，然后调用 select 函数将文件描述符集合拷贝到内核里，让内核来检查是否有网络事件产生，检查的方式很粗暴，就是通过遍历文件描述符集合的方式，当检查到有事件产生后，将此 Socket 标记为可读或可写， 接着再把整个文件描述符集合拷贝回用户态里，然后用户态还需要再通过遍历的方法找到可读或可写的 Socket，然后再对其处理。所以，对于 select 这种方式，需要进行 2 次「遍历」文件描述符集合，一次是在内核态里，一个次是在用户态里 ，而且还会发生 2 次「拷贝」文件描述符集合，先从用户空间传入内核空间，由内核修改后，再传出到用户空间中。select 使用固定长度的 BitsMap，表示文件描述符集合，而且所支持的文件描述符的个数是有限制的，在 Linux 系统中，由内核中的 FD_SETSIZE 限制， 默认最大值为 1024，只能监听 0~1023 的文件描述符。poll 不再用 BitsMap 来存储所关注的文件描述符，取而代之用动态数组，以链表形式来组织，突破了 select 的文件描述符个数限制，当然还会受到系统文件描述符限制。但是 poll 和 select 并没有太大的本质区别，都是使用「线性结构」存储进程关注的 Socket 集合，因此都需要遍历文件描述符集合来找到可读或可写的 Socket，时间复杂度为 O(n)，而且也需要在用户态与内核态之间拷贝文件描述符集合，这种方式随着并发数上来，性能的损耗会呈指数级增长。

>epoll
epoll ：操作系统提供了这三个函数。

第一步，创建一个 epoll 句柄
```java
int epoll_create(int size);
```
第二步，向内核添加、修改或删除要监控的文件描述符。
```java
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```
第三步，类似发起了 select() 调用
```java
int epoll_wait(int epfd, struct epoll_event *events, int max events, int timeout);
```


**epoll_create**
内核帮我们在epoll文件系统里建了个file结点；在内核cache里建了个红黑树用于存储以后epoll_ctl传来的socket；建立一个list链表，用于存储准备就绪的事件。

**epoll_ctl**
把socket放到epoll文件系统里file对象对应的红黑树上；给内核中断处理程序注册一个回调函数，告诉内核，如果这个句柄的中断到了，就把它放到准备就绪list链表里。

**epoll_wait**
观察list链表里有没有数据。有数据就返回，没有数据就sleep，等到timeout时间到后即使链表没数据也返回。而且，通常情况下即使我们要监控百万计的句柄，大多一次也只返回很少量的准备就绪句柄而已，所以，epoll_wait仅需要从内核态copy少量的句柄到用户态而已。

一颗红黑树，一张准备就绪句柄链表，少量的内核cache，解决了大并发下的socket处理问题。执行epoll_create时，创建了红黑树和就绪链表； 执行epoll_ctl时，如果增加socket句柄，则检查在红黑树中是否存在，存在立即返回，不存在则添加到树干上，然后向内核注册回调函数，用于当中断事件来临时向准备就绪链表中插入数据; 执行epoll_wait时立刻返回准备就绪链表里的数据即可。

1 select:在网络编程中统一的操作顺序是创建socket－>绑定端口－>监听－>accept->write/read,当有客户端连接到来时,select会把该连接的文件描述符放到fd_set（一组文件描述符(fd)的集合）,然后select会循环遍历它所监测的fd_set内的所有文件描述符，当select循环遍历完所有fd_set内指定的文件描述符对应的poll函数后，如果没有一个资源可用(即没有一个文件可供操作)，则select让该进程睡眠，一直等到有资源可用为止，fd_set是一个类似于数组的数据结构，由于它每次都要遍历整个数组，所有她的效率会随着文件描述符的数量增多而明显的变慢，除此之外在每次遍历这些描述符之前，系统还需要把这些描述符集合从内核copy到用户空间，然后再copy回去，如果此时没有一个描述符有事件发生（例如：read和write）这些copy操作和便利操作都是无用功，可见slect随着连接数量的增多，效率大大降低。可见如果在高并发的场景下select并不适用，况且select默认的最大描述符为1024，如果想要更多还要做响应参数的配置。

2 epoll：说到epoll都夸赞它的效率和并发量，那么她好在哪里呢。首先调用epoll_create时内核帮我们在epoll文件系统里建了个file结点；除此之外在内核cache里建立红黑树[红黑书介绍］(http://www.cnblogs.com/v-July-v/archive/2010/12/29/1983707.html)
用于存储以后epoll_ctl传来的socket，当有新的socket连接来时，先遍历红黑书中有没有这个socket存在，如果有就立即返回，没有就插入红黑数，然后给内核中断处理程序注册一个回调函数，每当有事件发生时就通过回调函数把这些文件描述符放到事先准备好的用来存储就绪事件的链表中，调用epoll_wait时，会把准备就绪的socket拷贝到用户态内存，然后清空准备就绪list链表，最后检查这些socket。在LT模式下，如果这些socket上确实有未处理的事件时，该句柄会再次被放回到刚刚清空的准备就绪链表，保证所有的事件都得到正确的处理。如果到timeout时间后链表中没有数据也立刻返回。因此在并发需求量高的场景中我们即使要监控数百万计的句柄，大多数一次也只返回很少量的准备就绪句柄。由此可见epoll仅需要从内核态copy少量的句柄到用户态，这样就避免了select模型中的无效便利和用户和内核之间的copy操作。

说道这里，可以看到epoll的优势非常明显，几乎没有描述符数量的限制，并发支持完美，不会随着socket的增加而降低效率，也不用在内核空间和用户空间之间做无效的copy操作。




**相对select的改进点**

内核中保存一份文件描述符集合，无需用户每次都重新传入，只需告诉内核修改的部分即可。
内核不再通过轮询的方式找到就绪的文件描述符，而是通过异步 IO 事件唤醒。（DMA技术的引入：DMA会回调/中断通知cpu io事件好了）
内核仅会将有 IO 事件的文件描述符返回给用户，用户也无需遍历整个文件描述符集合。

**epoll的LT模式（水平触发）和ET模式（边沿触发）**

LT模式和ET模式可以类比电平变化来学习，io读写的状态变化以socket为例有
- 可读：socket上有数据
- 不可读：socket上没有数据了
- 可写：socket上有空间可写
- 不可写：socket上无空间可写

[epoll的LT模式（水平触发）和ET模式（边沿触发）--解释的最清楚，没有之一](https://blog.csdn.net/albertsh/article/details/123958013)
>对于水平触发模式，一个事件只要有，就会一直触发。

>对于边缘触发模式，只有一个事件从无到有才会触发。且只触发一次

总结
- LT模式会一直触发EPOLLOUT，当缓冲区有数据时会一直触发EPOLLIN
- ET模式会在连接建立后触发一次EPOLLOUT，当收到数据时会触发一次EPOLLIN
- LT模式触发EPOLLIN时可以按需读取数据，残留了数据还会再次通知读取
- ET模式触发EPOLLIN时必须把数据读取完，否则即使来了新的数据也不会再次通知了
- LT模式的EPOLLOUT会一直触发，所以发送完数据记得删除，否则会产生大量不必要的通知
- ET模式的EPOLLOUT事件若数据未发送完需再次注册，否则不会再有发送的机会
- 通常发送网络数据时不会依赖EPOLLOUT事件，只有在缓冲区满发送失败时会注册这个事件，期待被通知后再次发送（水平触发的场景）

### 代码
```java
// ============bio====================
public class Server {
    public static void main(String[] args) throws IOException {
        ServerSocket ss = new ServerSocket();
        ss.bind(new InetSocketAddress("127.0.0.1", 8888));
        while(true) {
            Socket s = ss.accept(); //阻塞方法
            new Thread(() -> {
                handle(s);
            }).start();
        }
    }

    static void handle(Socket s) {
        try {
            byte[] bytes = new byte[1024];
            int len = s.getInputStream().read(bytes); //阻塞方法
            System.out.println(new String(bytes, 0, len));

            s.getOutputStream().write(bytes, 0, len); //阻塞方法
            s.getOutputStream().flush();
        } catch (IOException e) {
            e.printStackTrace();
        }

    }
}


// ============nio====================
public class Server {
  public static void main(String[] args) throws IOException {
    ServerSocketChannel ssc = ServerSocketChannel.open();
    ssc.socket().bind(new InetSocketAddress("127.0.0.1", 8888));
    ssc.configureBlocking(false);

    System.out.println("server started, listening on :" + ssc.getLocalAddress());
    Selector selector = Selector.open();
    ssc.register(selector, SelectionKey.OP_ACCEPT);

    while(true) {
      selector.select();
      Set<SelectionKey> keys = selector.selectedKeys();
      Iterator<SelectionKey> it = keys.iterator();
      while(it.hasNext()) {
        SelectionKey key = it.next();
        it.remove();
        handle(key);
      }
    }

  }

  private static void handle(SelectionKey key) {
    if(key.isAcceptable()) {
      try {
        ServerSocketChannel ssc = (ServerSocketChannel) key.channel();
        SocketChannel sc = ssc.accept();
        sc.configureBlocking(false);
        sc.register(key.selector(), SelectionKey.OP_READ );
      } catch (IOException e) {
        e.printStackTrace();
      } finally {
      }
    } else if (key.isReadable()) { //flip
      SocketChannel sc = null;
      try {
        sc = (SocketChannel)key.channel();
        ByteBuffer buffer = ByteBuffer.allocate(512);
        buffer.clear();
        int len = sc.read(buffer);

        if(len != -1) {
          System.out.println(new String(buffer.array(), 0, len));
        }

        ByteBuffer bufferToWrite = ByteBuffer.wrap("HelloClient".getBytes());
        sc.write(bufferToWrite);
      } catch (IOException e) {
        e.printStackTrace();
      } finally {
        if(sc != null) {
          try {
            sc.close();
          } catch (IOException e) {
            e.printStackTrace();
          }
        }
      }
    }
  }
}
```

## c10k问题
硬件资源（2GB 内存千兆网卡）
假设每个请求处理占用不到 200KB 的内存和 100Kbit 的网络带宽。，对于 2GB 内存千兆网卡的服务器，就可以满足并发 1 万个请求

软件资源（1万个线程，10G的内存）
假设一个线程占用1M内存（可设置），新到来一个 TCP 连接，就需要分配一个进程或者线程，那么如果要达到 C10K，意味着要一台机器维护 1 万个连接，相当于要维护 1 万个进程/线程，需要10G的内存，操作系统就算死扛也是扛不住的

## 参考文章-辅助理解
[嵌入式Linux驱动程序课程内容：I/O软件通常分为四个层次：用户空间I/O软件、设备独立性软件、设备驱动程序和中断处理程序，问以下各项工作是在哪个层次上完成的](https://blog.csdn.net/Albert_weiku/article/details/121315919)

[关于 I/O 要知道的那些事](jianshu.com/p/c3b109740287)

[epoll的LT模式（水平触发）和ET模式（边沿触发）](https://blog.csdn.net/albertsh/article/details/123958013)

[多路复用＋零拷贝--使用了多路复用的框架](https://new.qq.com/rain/a/20201117A0BFYF00)

[自己写的-io多路复用·零拷贝·while死循环&cpu](https://blog.csdn.net/qq_41063141/article/details/127003345)

[马士兵bio-nio-aio-netty代码demo](https://github.com/goodshred/NettyStudy)

[直观理解：Kafka零拷贝技术（Zero-Copy）](https://www.jianshu.com/p/0af1b4f1e164)

[零拷贝、如何实现零拷贝、大文件如何传输](https://blog.csdn.net/loytuls/article/details/123425947)

[从0到1打造高性能网络框架](https://blog.csdn.net/qq_41063141/article/details/129400860?csdn_share_tail=%7B%22type%22%3A%22blog%22%2C%22rType%22%3A%22article%22%2C%22rId%22%3A%22129400860%22%2C%22source%22%3A%22qq_41063141%22%7D)

[一些性能文章](https://heapdump.cn/article/5266573)