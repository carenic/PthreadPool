# PthreadPool
A Thread Pool  Realized by Pthread  
博客：https://blog.csdn.net/Gryphoon/article/details/98397581

## 简述
使用pthread实现的简单线程池，可以通过源码学习线程池的基本原理

## 使用
将文件 `PthreadPool.h` 和 `PthreadPool.cpp` 拷贝到需要使用的源码目录下，将头文件include在源码中，在编译时加上参数 `-lpthread` 即可

## 特性
* 使用C++开发
* 在创建时指定线程池中的线程数量  

## 类定义

``` C++
class PthreadPool
{
private:
    pthread_mutex_t lock;                             // 互斥锁
    pthread_cond_t notify;                            // 条件变量
    queue<Pthreadpool_Runable> thread_queue;          // 任务队列
    pthread_t *threads;                               // 任务数组
    int shutdown;                                     // 表示线程池是否关闭
    static void *threadpool_thread(void *threadpool); // 运行函数

public:
    PthreadPool();
    ~PthreadPool();
    int thread_num;                                                  // 线程数量
    int running_num;                                                 // 正在运行的线程数
    int waiting_num;                                                 // 队列中等待的数目
    int Init(unsigned int num);                                      // 初始化线程池
    int AddTask(void (*function)(void *), void *argument = nullptr); // 加入任务
    int Destory(PthreadPool_Shutdown flag = graceful_shutdown);      // 停止正在进行的任务并摧毁线程池
};
```

## 对外接口
```C++
int Init(unsigned int num);                                      // 初始化线程池
int AddTask(void (*function)(void *), void *argument = nullptr); // 加入任务
int Destory(PthreadPool_Shutdown flag = graceful_shutdown);      // 停止正在进行的任务并摧毁线程池
```
在使用前先创建对象 `PthreadPool pool;`  
然后再初始化线程池 `pool.Init(4);`  
在线程池中加入任务 `pool.AddTask(&dummy_task)`  
在程序结束时销毁线程池 `pool.Destory();`  
销毁线程池可以提供参数  
``` C++
enum PthreadPool_Shutdown {
    graceful_shutdown  = 1,   // 等待线程结束后关闭
    immediate_shutdown = 2  // 立即关闭
};
```  
## 测试运行
提供了一个简单的测试程序  
`g++ PthreadPool.cpp Test.cpp -lpthread -std=c++11 -o pthreadpool`

## *参考资料*
https://blog.csdn.net/jcjc918/article/details/50395528

-----------------------------------------------------------------------------------
使用 POSIX 消息队列在线程之间传递消息的示例代码：

``` C++
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <mqueue.h>
#include <unistd.h>

// 定义消息队列属性
struct mq_attr attr;
attr.mq_flags = 0;
attr.mq_maxmsg = 10;
attr.mq_msgsize = 128;
attr.mq_curmsgs = 0;

// 消息队列名称
const char* qname = "/myqueue";

// 线程函数
void* sender(void* arg) {
    char message[128];
    sprintf(message, "Message from thread %ld", pthread_self());
    mqd_t mq = mq_open(qname, O_CREAT | O_WRONLY, 0644, &attr);
    mq_send(mq, message, strlen(message) + 1, 0);
    mq_close(mq);
    return NULL;
}

void* receiver(void* arg) {
    mqd_t mq = mq_open(qname, O_RDONLY);
    char message[128];
    mq_receive(mq, message, sizeof(message), NULL);
    printf("Received: %s\n", message);
    mq_close(mq);
    mq_unlink(qname);
    return NULL;
}

int main() {
    pthread_t ts, tr;
    pthread_create(&ts, NULL, sender, NULL);
    sleep(1); // 确保发送者先启动
    pthread_create(&tr, NULL, receiver, NULL);
    pthread_join(ts, NULL);
    pthread_join(tr, NULL);
    return 0;
}
```  
需要说明的是：POSIX 消息队列 (mqueue) 的数据类型并不局限于字符类型。在 POSIX 消息队列中，消息可以是任意数据类型，只要满足以下条件：

- 消息大小：消息的大小不能超过在消息队列创建时指定的最大消息大小 (mq_msgsize)。

- 消息类型：消息通常是一个字节序列，可以是结构体、字符数组、整数、浮点数等任何数据类型。但是，发送到消息队列的数据需要是连续的内存区域，因为 mq_send() 函数需要一个指向数据的指针。

- 消息结构：通常，为了发送复杂的数据结构，我们会将数据结构序列化为一个二进制格式的字节序列（例如通过谷歌开源库 protobuf）然后发送它。接收方再将这个二进制数组反序列化回原始数据结构。

下面是一个发送结构体作为消息的例子：

```C++
#include <mqueue.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "mydata.pb.h"  // 引入 Protobuf 生成的头文件

int main() {
    mqd_t mq;
    struct mq_attr attr;
    mypackage::MyData data;
    std::string buffer; // 用于存储序列化的数据

    // 初始化消息队列属性
    attr.mq_flags = 0;
    attr.mq_maxmsg = 10;
    attr.mq_msgsize = 256; // 设置足够大的大小
    attr.mq_curmsgs = 0;

    // 创建消息队列
    mq = mq_open("/myqueue", O_CREAT | O_WRONLY, 0644, &attr);
    if (mq == (mqd_t)-1) {
        perror("mq_open");
        exit(1);
    }

    // 填充数据
    data.set_id(1);
    data.set_value(3.14f);
    data.set_name("Sample");

    // 序列化数据
    if (!data.SerializeToString(&buffer)) {
        perror("SerializeToString");
        exit(1);
    }

    // 发送消息
    if (mq_send(mq, buffer.data(), buffer.size(), 0) == -1) {
        perror("mq_send");
        exit(1);
    }

    // 关闭消息队列
    mq_close(mq);
    return 0;
}

// 接收消息
void *receiver(void *arg) {
    mqd_t mq;
    struct mq_attr attr;
    std::string buffer; // 用于存储接收的数据
    mypackage::MyData data;

    // 打开消息队列
    mq = mq_open("/myqueue", O_RDONLY);
    if (mq == (mqd_t)-1) {
        perror("mq_open");
        exit(1);
    }

    // 接收消息
    buffer.resize(256); // 确保足够大的大小
    if (mq_receive(mq, &buffer[0], buffer.size(), NULL) == -1) {
        perror("mq_receive");
        exit(1);
    }

    // 反序列化数据
    if (!data.ParseFromString(buffer)) {
        perror("ParseFromString");
        exit(1);
    }

    // 打印接收到的数据
    printf("Received: id=%d, value=%f, name=%s\n", data.id(), data.value(), data.name().c_str());

    // 关闭消息队列
    mq_close(mq);
    mq_unlink("/myqueue");

    return NULL;
}
```

要想使用上述代码，首先需要定义一个 .proto 文件(在其中定义消息格式)，然后使用 protoc 编译器生成 C++ 文件 mydata.pb.h 和 mydata.pb.cc (其中包含了 Protobuf 类的定义和实现)。

使用 Protocol Buffers (Protobuf) 进行序列化和反序列化相比 memcpy() 有以下优势和不足：

优势：

1.跨平台兼容性：

- Protobuf 生成的序列化代码可以在不同的操作系统和硬件架构之间保持一致，确保数据的兼容性。
- 与之相比，使用 memcpy() 序列化和反序列化时，如果不同的平台对数据类型的大小端有不同的解释，可能会导致不兼容。

2.可扩展性：

- Protobuf 允许你修改 .proto 文件中的消息定义而不会破坏已有的系统，因为 Protobuf 支持向后兼容。
- 与之相比，使用 memcpy() 序列化和反序列化时，如果结构体定义发生变化，可能需要修改所有使用该结构体的代码。

3.高效的存储和传输：

- Protobuf 使用变长编码，使得小整数使用更少的字节，这可以减少数据的存储和传输大小。
- 与之相比，memcpy() 会复制整个结构体，包括所有的填充字节，可能导致数据膨胀。

4. 类型安全：

- Protobuf 序列化时会检查字段类型，确保数据的一致性。
- memcpy() 不进行类型检查，如果结构体定义错误，可能会导致数据损坏。

5.自动生成代码：

- Protobuf 提供了代码生成器，可以自动生成序列化和反序列化的代码，减少了手动编写和维护这些代码的工作量。
- 使用 memcpy() 需要手动编写序列化和反序列化的代码。

6. 丰富的库支持：

- Protobuf 有丰富的库支持，包括各种编程语言的实现，方便在不同的系统中使用。
- memcpy() 依赖于底层的 C/C++ 语言，可能需要额外的工作来在其他语言中实现类似的功能。

不足：

1.学习曲线：

- Protobuf 需要学习 .proto 文件的语法和使用 protoc 编译器，对于新手来说可能有一定的学习成本。
- memcpy() 在 C/C++ 中非常常见，大多数开发者都很熟悉。
 
2.复杂性：

- 对于非常简单的数据结构，使用 Protobuf 可能会引入不必要的复杂性。
- memcpy() 对于简单的数据复制任务来说非常直接和简单。

3.性能：

- 虽然 Protobuf 的性能通常很好，但在某些极端的性能敏感场景下，手动优化的 memcpy() 可能会更快。
- Protobuf 的序列化和反序列化过程涉及额外的计算，如变长编码和字段编号的解析。

4.二进制格式的可读性：

- Protobuf 序列化后的二进制格式不是人类可读的，这可能会使得调试和测试更加困难。
- 结构体可以通过内存转储工具以人类可读的格式查看。

5.依赖性：

- 使用 Protobuf 意味着你的项目依赖于 Protobuf 库和工具链。
- memcpy() 不需要任何额外的依赖。

总结来说，Protobuf 提供了一种结构化、高效、跨平台的数据序列化方法，适合复杂的数据交换场景。而 memcpy() 更适合简单的、性能敏感的或者不需要跨平台兼容的场景。在选择序列化方法时，需要根据项目的具体需求和上下文来决定使用哪种方法。
