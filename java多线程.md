# 多线程 与 操作系统 与 网络通信


用户态和内核态，计算机底层的运行代码，操作硬件的内核代码，防止外部代码的恶意攻击，维持系统的稳定性。
应用程序在涉及到计算机内部资源的调用（需要操作硬件）时，表现为调用内核代码，由用户态转换为内核态，页码寻址（相较于普通的线程切换《上下文》更加耗时的原因）找到内核内存，线程切换，cpu在指定内存中开始运行相关代码，用户代码阻塞/非阻塞；
计算机资源的存储与访问 ; 本地的磁盘文件IO ; 网络通信IO;

IO操作，涉及到文件的操作但是不涉及应用程序层面的交互，比如控制台的输出，外部网络通信传输等等，我们可以通过MappedByteBuffer直接内存映射文件 在FileChannel中 将操作系统分配给JVM用户对用户直接内存的操作权来执行指令操作；省去内核缓存区向JVM用户缓存区的数据拷贝；一次性获取channel中的文件长度对应的字节数组，然后put get指令进行；


************************************* 手动实现复制transfer() ********************************
*      try (FileChannel sourceChannel = FileChannel.open(Paths.get("logs/javabetter/itwanger.txt"), StandardOpenOption.READ);
       FileChannel destinationChannel = FileChannel.open(Paths.get("logs/javabetter/itwanger3.txt"), StandardOpenOption.WRITE, StandardOpenOption.CREATE, StandardOpenOption.READ)) {
       sourceChannel.transferTo(0, sourceChannel.size(), destinationChannel);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }  
*          try (FileChannel sourceChannel = FileChannel.open(Paths.get("logs/javabetter/itwanger.txt"), StandardOpenOption.READ);
              FileChannel destinationChannel = FileChannel.open(Paths.get("logs/javabetter/itwanger2.txt"), StandardOpenOption.WRITE, StandardOpenOption.CREATE, StandardOpenOption.READ)) {
        
            long fileSize = sourceChannel.size();
            MappedByteBuffer sourceMappedBuffer = sourceChannel.map(FileChannel.MapMode.READ_ONLY, 0, fileSize);
            MappedByteBuffer destinationMappedBuffer = destinationChannel.map(FileChannel.MapMode.READ_WRITE, 0, fileSize);
        
            for (int i = 0; i < fileSize; i++) {
                byte b = sourceMappedBuffer.get(i);
                destinationMappedBuffer.put(i, b);
            }
            destinationMappedBuffer.force(); // 强制刷回磁盘
        }

阶段 1：前期授权（内核做）  
在执行循环前，你调用的 sourceChannel.map(...) 会触发：  
JVM 调用底层的本地方法（JNI），向内核发起 mmap() 系统调用；  
内核为 JVM 进程分配一块「用户态直接内存」，并建立 “内存地址 ↔ 磁盘文件” 的映射关系；  
内核把这块内存的访问地址和权限返回给 JVM；  
JVM 基于这个地址创建 MappedByteBuffer 对象（相当于给这块内存贴了个 “操作接口”）。  
阶段 2：循环读写（JVM 做，内核不参与）  
循环里的 get(i)/put(i) 是纯 JVM 操作：  
sourceMappedBuffer.get(i)：  
JVM 通过 MappedByteBuffer 持有的内存地址，直接访问「源直接内存」的第 i 个字节；  
把这个字节的值读取到 JVM 栈的局部变量 b 中（这一步是 “直接内存 → JVM 栈” 的临时拷贝，属于用户态内部操作）；  
destinationMappedBuffer.put(i, b)：  
JVM 把栈里的 b 写入「目标直接内存」的第 i 个字节；  
全程没有触发任何内核系统调用，内核甚至 “不知道” JVM 在读写这块内存 —— 就像你拿到物业授权后，自己进出公共储物间搬东西，物业不用全程参与。  
阶段 3：数据同步到磁盘（内核做，JVM 可选触发）  
循环执行完后：  
目标直接内存里已经存了拷贝后的数据，但还在内存中，没写到磁盘；  
内核会通过「页缓存机制」，在合适的时机（比如内存不足、定时刷盘）自动把直接内存的数据同步到磁盘；  
你也可以调用 destinationMappedBuffer.force()，让 JVM 通知内核 “立即刷盘”（触发 msync() 系统调用）。  


网络通信指通信的双方通过计算机网卡实现信息的传递，底层表现为将磁盘中信息通过线程CPU在硬件分配给程序的内存按照程序指令进行封装，从网卡的指定端口在网络通信协议下进行发送和接收，（触发内核代码）。
网络通信之连接，InetAdress()类来封装new InetAdress【网络中的端口信息（网络IP , 端口号） 】 网络IP -> 计算机回环地址（负责内部通信 映射为回环网卡） 网络（局域）地址（负责网络内通信 物理网卡） 虚拟机（虚拟网卡的虚拟地址）
IP表示：域名NDS eg: 本地的"localhost" 实际IPV4/IPV6 eg:127.0.0.1表示回环地址

套接字：socket,DatagramSocket作为应用层 与 TCP|UDP/IP 协议族通信的中间软件抽象层，表现为一个封装了 TCP|UDP / IP协议族 的编程接口（API），
ServerSocket：服务端的 “监听接口”（只负责等客户端来连接，不直接收发数据）监听fd是否就绪，就绪后转化为读状态就绪，抽象出Socket clientSocket = serverSocket.accept()后续全部交给socket处理；
Socket / DatagramSocket ：客户端 / 服务端的 “通信接口”（连接建立后，真正用来发 / 收数据的对象）。


应用层实现http协议，避免由于TCP字节流协议可能出现的粘包问题，http协议规定接收双方应用层程序需要按照协议规则进行封装和拆解，得到主要的字段信息；
抽象Socket选择传输协议，进行字段的对应封包；
TCP为控制数据传输的稳定性，安全性与效率，通过滑动窗口，ACK包确认机制，缓存区滑动窗口（以发送未确认，可发送未发送），Nagle算法（触发发包的具体规则，控制发包的大小(合并，拆解1460）和时间（200ms)）保证接收双方






