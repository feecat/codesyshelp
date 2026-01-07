# 13 附录：并发控制与通信机制

## 13.1 简介

在C/C++项目开发中，多线程技术的应用十分广泛，典型的如Qt应用中的UI主线程与后台工作线程的分离。而在复杂的系统架构中，不仅同一程序的线程间需要协作，不同进程间也往往需要进行数据交换。

实现并发控制与通信的手段多种多样，基础层面的手段包括**原子变量（Atomic）、互斥锁（Mutex）**，以及基于RAII机制的锁封装（如 **`std::unique_lock`、`std::lock_guard`**）；进阶的同步机制还包括**条件变量（Condition Variable）、读写锁（Read-Write Lock）以及信号量（Semaphore）**等。

虽然工具繁多，但在实际工程实践中，为了平衡性能与开发维护成本，推荐采用以下策略：

*   **线程间通讯 (Inter-Thread)**
    *   **推荐方案：** 自旋锁 + 内存拷贝 (Lock & Copy)
    *   **适用场景：** 通用数据交互，逻辑简单且安全。
*   **进程间通讯 (Inter-Process)**
    *   **推荐方案：** ZeroMQ (ZMQ) 及其 `ZMQ_CONFLATE` 选项
    *   **适用场景：** 解耦的模块间通讯。`ZMQ_CONFLATE` 特别适用于只需保留最新数据的遥测/状态更新场景。
*   **超低延迟/实时通讯 (Real-Time)**
    *   **推荐方案：** 共享内存 (Shared Memory) + 序列锁 (SeqLock) + 内存屏障 (Memory Barrier)
    *   **适用场景：** 伺服控制、高频信号处理等对延迟极度敏感的场景。

## 13.2 不加锁的风险与后果

其实，不加锁在大部分情况下都运行的很好，比如：

```cpp
线程A：
case 10:
    g_var = 99;
//some code
case 80:
    g_var = 0;
	
线程B：
if (g_var == 99){
    //do something
}
```

这尤其适用于一个线程仅写、另一个线程仅读的情况，且对时序不敏感。对于内存对齐的简单类型（如int、bool）的赋值操作通常是原子的。在这种“一写多读”且不涉及复杂计算的场景下，不加锁确实可以避免崩溃，偶尔只会因为缓存一致性延迟导致读取稍微慢一点。

然而，在实际工程中，更多的情况是多个线程同时对同一个变量进行“读取-修改-写入”的操作。这种情况下不加锁，会引发竞态条件（Race Condition），导致严重的数据错误。例如：

```cpp
// 线程A
for (int i = 0; i < 10000; ++i) {
    g_count++; 
}

// 线程B
for (int i = 0; i < 10000; ++i) {
    g_count++;
}
```

很明显`g_count`在这里就被两个线程竞争写入了，但还有一些不是那么明显的场景，例如：
```cpp
// 线程A
if (something)
    axis1.control_word = 0x15;
	//some code
	axis1.target_position = 10000;

// 线程B
if (axis1.control_word == 0x15)
    slave.target_position = axis1.target_position;
```
这时，就变成有一定的概率线程A先写了control_word，还没来得及修改target_position，此时线程B拿到的就是旧值。  
除此之外，还有一些更隐蔽的，例如：
```cpp
// 线程A
lock = true;
g_count = 1000;
lock = false;

// 线程B
if (!lock){
    output = g_count;
}
```
乍一看起来，线程A用lock锁住了变量，线程B读取前检验没有问题，但实际跑的时候就会出现output=1000。有几种原因可以造成问题：

1. 线程B先判断lock=false，通过了，此时线程A写了lock = true。
2. 在多核心CPU上，读和写操作发生在同一时刻或相近时刻，但线程A操作的lock变量还在CPU缓存中，并没有更新到内存。
3. 编译器可能会进行优化，将lock直接写false。（一般o2及以内不会）
4. CPU可能会乱序执行，但压栈出栈顺序还是编译出来程序的顺序，一般也不会导致问题。

因此，在架构一个程序时，一旦牵涉到多线程，我们应当使用一些措施避免冲突，且应尽可能简化这一过程。

## 13.3 线程间通讯：自旋锁+内存拷贝

自旋锁（Spinlock）是一种低开销的同步机制，特别适合临界区极短（例如仅做一次内存拷贝）的场景。与传统的互斥锁（Mutex）不同，自旋锁在获取不到锁时不会让线程进入睡眠（上下文切换），而是让CPU“空转”等待，从而避免了昂贵的系统调用开销。

在单写单读的场景下，结合 std::atomic_flag 和 memcpy，我们可以实现高效且数据安全的数据交换。以下是一个完整的测试示例：

```
#include <QCoreApplication>
#include <QThread>
#include <QDebug>
#include <atomic>

struct globalData{
    uint32_t data1;
    uint32_t data2;
};

std::atomic_flag g_dataLock = ATOMIC_FLAG_INIT;

globalData s1Data = {0, 0xFFFF};
globalData s2Data = {0, 0xFFFF};
globalData g_data = {0, 0xFFFF};

bool enable_lock = true;

void busy_wait(int loops) {
    for (volatile int i = 0; i < loops; ++i) {}
}

void process1() {
    while(true) {
        QThread::msleep(1);
        if (enable_lock) {
            s1Data.data1++;
            busy_wait(100);
            s1Data.data2 = 0xFFFF - s1Data.data1;

            if (!g_dataLock.test_and_set(std::memory_order_acquire)) {
                memcpy(&g_data, &s1Data, sizeof(g_data));
                g_dataLock.clear(std::memory_order_release);
            }
        }
        else {
            g_data.data1++;
            busy_wait(100);
            g_data.data2 = 0xFFFF - g_data.data1;
        }

        QThread::yieldCurrentThread();
    }
}

void process2() {
    while(true) {
        QThread::msleep(1);
        if (enable_lock) {
            while (g_dataLock.test_and_set(std::memory_order_acquire)) {
                QThread::yieldCurrentThread();
            }
            memcpy(&s2Data, &g_data, sizeof(s2Data));
            g_dataLock.clear(std::memory_order_release);
        }
        else {
            memcpy(&s2Data, &g_data, sizeof(s2Data));
        }

        if (s2Data.data2 != (0xFFFF - s2Data.data1)) {
            qCritical() << "!!! DATA TEARING DETECTED !!!";
            qDebug() << "data1:" << s2Data.data1
                     << "data2:" << s2Data.data2
                     << "Expected data2:" << (0xFFFF - s2Data.data1);
        }
    }
}

int main(int argc, char *argv[])
{
    QCoreApplication a(argc, argv);

    qDebug() << "Test Started. Lock Enabled:" << enable_lock;

    QThread *thread1 = QThread::create(process1);
    QThread *thread2 = QThread::create(process2);

    thread1->start();
    thread2->start();

    return a.exec();
}
```

在这个示例中，我们使用了`std::atomic_flag`来实现自旋锁，这是C++标准库中唯一保证是lock-free（无锁实现）的原子类型。  
加锁：`g_dataLock.test_and_set(std::memory_order_acquire)`。如果返回 false，说明抢锁成功；如果返回 true，说明锁已被占用。  
解锁：`g_dataLock.clear(std::memory_order_release)`。  
你可以切换`enable_lock`的初始值并运行一下。

## 13.4 进程间通讯：ZeroMQ

ZeroMQ (ZMQ) 是一个极高性能的异步消息通信库，它将复杂的 Socket 编程抽象为简洁的消息队列模型。与传统的 TCP/UDP 编程相比，ZMQ 最大的优势在于它是**面向消息（Message-oriented）**的：它自动处理了底层连接断开重连、消息分帧（Framing）以及粘包/拆包问题。

以下是使用 ZMQ PUSH/PULL 模式配合 ZMQ_CONFLATE 选项实现的双向通讯示例。该方适用于**“只关心最新状态”**（如传感器实时读数、UI显示）的场景，它允许一定程度的丢帧，但只要主从程序正常运行，总是能读到最新的数据。

```
//master.cpp
#include <cstdint>
#include <vector>
#include <chrono>
#include <iostream>
#include <zmq.h>
#include <unistd.h>

const size_t PACKET_SIZE = 1024; // 1 KByte
const char* ADDR_MASTER_OUT = "ipc:///tmp/zmq_master.ipc";
const char* ADDR_SLAVE_OUT  = "ipc:///tmp/zmq_slave.ipc";

int main() {
    void* ctx = zmq_ctx_new();
    void* tx = zmq_socket(ctx, ZMQ_PUSH);
    zmq_bind(tx, ADDR_MASTER_OUT);

    void* rx = zmq_socket(ctx, ZMQ_PULL);
    zmq_connect(rx, ADDR_SLAVE_OUT);

    int conflate = 1;
    zmq_setsockopt(rx, ZMQ_CONFLATE, &conflate, sizeof(conflate));

    std::vector<uint8_t> tx_buf(PACKET_SIZE, 0);
    std::vector<uint8_t> rx_buf(PACKET_SIZE, 0);
    
    //ConnectionWatchdog watchdog;

    std::cout << "[Master] Started. TX bound to " << ADDR_MASTER_OUT << std::endl;

    while (true) {
        tx_buf[0]++;
        zmq_send(tx, tx_buf.data(), PACKET_SIZE, ZMQ_DONTWAIT);

        int len = zmq_recv(rx, rx_buf.data(), PACKET_SIZE, ZMQ_DONTWAIT);
        
        if (len > 0) {
            printf("[Master] Recv: %02X | Send: %02X\n", rx_buf[0], tx_buf[0]);
        }
        usleep(10000); 
    }

    zmq_close(tx);
    zmq_close(rx);
    zmq_ctx_destroy(ctx);
    return 0;
}


//slave.cpp
#include <cstdint>
#include <vector>
#include <chrono>
#include <iostream>
#include <zmq.h>
#include <unistd.h>

const size_t PACKET_SIZE = 1024; // 1 KByte
const char* ADDR_MASTER_OUT = "ipc:///tmp/zmq_master.ipc";
const char* ADDR_SLAVE_OUT  = "ipc:///tmp/zmq_slave.ipc";

int main() {
    void* ctx = zmq_ctx_new();
    void* tx = zmq_socket(ctx, ZMQ_PUSH);
    zmq_bind(tx, ADDR_SLAVE_OUT);

    void* rx = zmq_socket(ctx, ZMQ_PULL);
    zmq_connect(rx, ADDR_MASTER_OUT);

    int conflate = 1;
    zmq_setsockopt(rx, ZMQ_CONFLATE, &conflate, sizeof(conflate));

    std::vector<uint8_t> tx_buf(PACKET_SIZE, 0xA0);
    std::vector<uint8_t> rx_buf(PACKET_SIZE, 0);

    //ConnectionWatchdog watchdog;

    std::cout << "[Slave] Started. TX bound to " << ADDR_SLAVE_OUT << std::endl;

    while (true) {
        int len = zmq_recv(rx, rx_buf.data(), PACKET_SIZE, ZMQ_DONTWAIT);
        
        if (len > 0) {
            printf("[Slave]  Recv: %02X | Send: %02X\n", rx_buf[0], tx_buf[0]);
            
            tx_buf[0]++;
            zmq_send(tx, tx_buf.data(), PACKET_SIZE, ZMQ_DONTWAIT);
        }
        usleep(1000); 
    }

    zmq_close(tx);
    zmq_close(rx);
    zmq_ctx_destroy(ctx);
    return 0;
}

```

ZMQ的使用是很简单的，文档也很多。需要注意的是，虽然延迟很低（通常在微秒级），但ZMQ不是为实时通讯设计的协议，它可能会存在动态内存分配，不建议在EtherCAT等高实时性线程中使用ZMQ。

## 13.5 进程间通讯：共享内存

共享内存本身的实现很简单高效，两个程序都可以读写同一段内存地址，它以文件的形式展示在`/dev/shm/`中。以下是初始化部分的代码：

```cpp
int fdData = shm_open(sSharedMemName, O_CREAT | O_RDWR, S_IRWXU | S_IRWXG);
ftruncate(fdData, sizeof(*sData));
sData = (sDataStruct*)mmap(0, sizeof(*sData), PROT_READ | PROT_WRITE, MAP_SHARED, fdData, 0);
close(fdData);
```

但是，共享内存也是会有冲突问题的，解决的方法也很多，属于“群魔乱舞”了。我们这里使用共享内存 (Shared Memory) + 序列锁 (SeqLock) + 内存屏障 (Memory Barrier)的方式：

- 共享内存：作为基础的数据传输介质，实现数据的零拷贝传输。
- 序列锁：利用版本号机制解决读写冲突，确保读者在读取数据期间，写者未对数据进行修改。
- 内存屏障：强制约束 CPU 和编译器的指令排序，防止因乱序导致的数据不一致。

```cpp
//master.cpp
#include <iostream>
#include <atomic>
#include <cstring>
#include <cmath>
#include <thread>
#include <chrono>
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

struct SharedData {
    uint32_t sequence; 
    double timestamp;
    double values[16];
    uint32_t status_code;
};

const char* SHM_NAME = "demo_shm";
SharedData* data;

int main() {
	int fdData = shm_open(SHM_NAME, O_CREAT | O_RDWR, S_IRWXU | S_IRWXG);
	ftruncate(fdData, sizeof(*data));
	data = (SharedData*)mmap(0, sizeof(*data), PROT_READ | PROT_WRITE, MAP_SHARED, fdData, 0);
	close(fdData);

    std::cout << "[Master] Started. Writing to " << SHM_NAME << std::endl;
	data->sequence = 0;
    long long counter = 0;
    while (true) {
        double current_time = std::chrono::duration_cast<std::chrono::milliseconds>(
            std::chrono::system_clock::now().time_since_epoch()
        ).count() / 1000.0;
        
        counter++;

        volatile uint32_t* seq_ptr = &data->sequence;
        uint32_t seq = *seq_ptr;
        *seq_ptr = seq + 1;
        std::atomic_thread_fence(std::memory_order_seq_cst);
        data->timestamp = current_time;
        data->status_code = (counter % 100);
        for (int i = 0; i < 16; ++i) {
            data->values[i] = std::sin(current_time + i * 0.5);
        }
		//or, you can use memcpy to reduce process time
        std::atomic_thread_fence(std::memory_order_seq_cst);
        *seq_ptr = seq + 2;

        if (counter % 100 == 0) {
            std::cout << "[Master] Updated Seq: " << *seq_ptr 
                      << " | TS: " << data->timestamp << std::endl;
        }

        std::this_thread::sleep_for(std::chrono::milliseconds(10)); // 100Hz
    }

    return 0;
}


//slave.cpp
#include <iostream>
#include <atomic>
#include <cstring>
#include <thread>
#include <chrono>
#include <iomanip>
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

struct SharedData {
    uint32_t sequence; 
    double timestamp;
    double values[16];
    uint32_t status_code;
};

const char* SHM_NAME = "demo_shm";
SharedData* data;

int main() {
	int fdData = shm_open(SHM_NAME, O_CREAT | O_RDWR, S_IRWXU | S_IRWXG);
	ftruncate(fdData, sizeof(*data));
	data = (SharedData*)mmap(0, sizeof(*data), PROT_READ | PROT_WRITE, MAP_SHARED, fdData, 0);
	close(fdData);

    SharedData local_copy;

    std::cout << "[Slave] Attached. Reading..." << std::endl;
	uint32_t seq_old = data->sequence;
    while (true) {
        uint32_t seq1, seq2;
        volatile uint32_t* seq_ptr = &data->sequence;

		seq1 = *seq_ptr;
		std::atomic_thread_fence(std::memory_order_seq_cst);
		memcpy(&local_copy, data, sizeof(local_copy));
		std::atomic_thread_fence(std::memory_order_seq_cst);
		seq2 = *seq_ptr;
        
        if (!(seq1 & 1) && (seq1 == seq2) && (seq1 != seq_old)){
			seq_old = seq1;
			
			std::cout << std::fixed << std::setprecision(3);
			std::cout << "[Slave] Seq: " << seq2 
					  << " | Status: " << local_copy.status_code 
					  << " | Time: " << local_copy.timestamp 
					  << " | Val[0]: " << local_copy.values[0] 
					  << std::endl;
		}
        std::this_thread::sleep_for(std::chrono::milliseconds(2));
    }

    return 0;
}

```

需要注意的几个点是：

- 使用`volatile uint32_t* seq_ptr = &data->sequence;`来避免编译器优化
- 使用`std::atomic_thread_fence(std::memory_order_seq_cst);`强制按序执行
- 校验读取前后`sequence`奇偶和一致性

## 13.6 结尾

回顾本章，针对不同场景我们推荐了三种务实的工程策略：

1. 线程间通讯：首选 “自旋锁 + 内存拷贝”。它简单、安全且高效，足以应付绝大多数场景。
2. 进程间通讯：首选 ZeroMQ。它能极好地解耦业务逻辑，特别是 ZMQ_CONFLATE 模式。
3. 极低延迟场景：需要实时性可考虑 “共享内存 + 序列锁”，所有编程语言都支持。

不过，对于一些只读且对时序不敏感的应用，如UI更新，仍然可以采用非锁实现。