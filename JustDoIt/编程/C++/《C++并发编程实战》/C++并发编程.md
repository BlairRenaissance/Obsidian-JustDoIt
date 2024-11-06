
注：并不是所有的情况都应该用并发，一个线程大约占1m的空间，以及会带来线程切换的代价

compare swap 原子操作

# 管理线程

## 基本线程管理

```
// join
std::thread t(do_work);
t.join();
// detach
std::thread t(do_background_work);
t.detach();
```

## 将参数传递给线程函数

```
// char *
void f(int i,std::string const& s);
void not_oops(int some_param)
{
  char buffer[1024];  sprintf(buffer,"%i",some_param);  std::thread t(f,3,std::string(buffer));  t.detach();
}

// 引用
std::thread t(update_data_for_widget,w,std::ref(data));

// 成员函数
class X
{
public:
    void do_lengthy_work();
};
X my_x;
std::thread t(&X::do_lengthy_work,&my_x);
// can’t be copied but can only be moved
void process_big_object(std::unique_ptr<big_object>);
std::unique_ptr<big_object> p(new big_object);
p->prepare_data(42);
std::thread t(process_big_object,std::move(p));
```

## 转移线程的所有权

```
void some_function();
void some_other_function();
std::thread t1(some_function);
std::thread t2=std::move(t1);
t1=std::thread(some_other_function);
std::thread t3;
t3=std::move(t2);
t1=std::move(t3);
```

## 在运行时选择线程数量

```
std::thread::hardware_ concurrency()
```

## 识别线程

```
std::this_thread:: get_id()
```

# 线程间共享数据

if all shared data is read-only, there’s no problem.

## 使用互斥锁保护共享数据

### Using mutexes in C++

```
#include <list>
#include <mutex>
#include <algorithm>
std::list<int> some_list;
std::mutex some_mutex;
void add_to_list(int new_value)
{
	std::lock_guard<std::mutex> guard(some_mutex);
	some_list.push_back(new_value);
}
```

### 不要将指向受保护数据的指针和引用传递到锁的范围之外

### 发现接口中固有的竞态条件

### 死锁：问题和解决方案
lock the two mutexes in the same order

```
friend void swap(X& lhs, X& rhs)
{
	if(&lhs==&rhs)    	return;    
	std::lock(lhs.m,rhs.m);    
	std::lock_guard<std::mutex> lock_a(lhs.m,std::adopt_lock);
	std::lock_guard<std::mutex> lock_b(rhs.m,std::adopt_lock);
	swap(lhs.some_detail,rhs.some_detail);
}
// c++17
std::scoped_lock<>
```

### 更多避免死锁的指南

- 避免嵌套锁
- 避免在持有锁的情况下调用用户提供的代码
- 使用锁层次结构

```
hierarchical_mutex high_level_mutex(10000);
hierarchical_mutex low_level_mutex(5000);
hierarchical_mutex other_mutex(6000);
```

### Flexible locking with std::unique_lock

### Transferring mutex ownership between scopes

```
std::unique_lock<std::mutex> get_lock()
{
    extern std::mutex some_mutex;    
    std::unique_lock<std::mutex> lk(some_mutex);    
    prepare_data();    
    return lk;
}
void process_data()
{
    std::unique_lock<std::mutex> lk(get_lock());    
    do_something();
}
```

### 以适当的粒度进行锁定

- 一般来说，只应在执行所需操作的最短可能时间内持有锁。
- 如果复制成本低，你不应该使用锁。

## Alternative facilities for protecting shared data

### Protecting shared data during initialization

```
// std::once_flag
std::shared_ptr<some_resource> resource_ptr;
std::once_flag resource_flag;

void init_resource()
{
    resource_ptr.reset(new some_resource);
}

void foo() {
    std::call_once(resource_flag,init_resource);    
    resource_ptr->do_something();
}

// a local variable declared with static.
class my_class;
my_class& get_my_class_instance()
{
    static my_class instance;    
    return instance;
}
```

### Protecting rarely updated data structures

```
#include <map>
#include <string>
#include <mutex>
#include <shared_mutex>

class dns_entry;
class dns_cache
{
    std::map<std::string,dns_entry> entries;    
    mutable std::shared_mutex entry_mutex;    
public:
	dns_entry find_entry(std::string const& domain) const {
		std::shared_lock<std::shared_mutex> lk(entry_mutex);
		std::map<std::string,dns_entry>::const_iterator const it =       entries.find(domain);    
		return (it==entries.end())?dns_entry():it->second;
	}
	
	void update_or_add_entry(std::string const& domain, dns_entry const& dns_details){
	    std::lock_guard<std::shared_mutex> lk(entry_mutex);
	    entries[domain] = dns_details;
	}
};
```

### 递归锁

但不推荐这样的使用，因为它可能导致思维散漫和设计不良。

特别是，当持有锁时，类不变量通常会被破坏，这意味着即使在不变量被破坏的情况下调用，第二个成员函数也需要工作。

# 同步并发操作

## 等待事件或其他条件

首先，它可以在共享数据（由互斥锁保护）中不断检查一个标志，并在第二个线程完成任务时设置该标志。

第二个选项是让等待线程在检查之间使用std::this_thread::sleep_for()函数短暂休眠。

第三个也是首选的选项是使用C++标准库的设施来等待事件本身。

### Waiting for a condition with condition variables

```
std::mutex mut;
std::queue<data_chunk> data_queue;
std::condition_variable data_cond;
void data_preparation_thread()
{
    while(more_data_to_prepare())
    {
      data_chunk const data=prepare_data();
      {
          std::lock_guard<std::mutex> lk(mut);
          data_queue.push(data);
      }
      data_cond.notify_one();
    }
}

void data_processing_thread()
{
    while(true)
    {
        std::unique_lock<std::mutex> lk(mut);
        data_cond.wait(lk,[]{return !data_queue.empty();});
        data_chunk data=data_queue.front();
        data_queue.pop();
        lk.unlock();
        process(data);
        if(is_last_chunk(data))
        break;
	}
}
```

## Waiting for one-off events with futures

```
std::future<>
std::shared_future<>
std::experimental::future<>
std::experimental::shared_future<>
```

### Returning values from background tasks

```
#include <future>
#include <iostream>
int find_the_answer_to_ltuae();
void do_other_stuff();
int main()
{
    std::future<int> the_answer=std::async(find_the_answer_to_ltuae);
    do_other_stuff();
    std::cout<<"The answer is "<<the_answer.get()<<std::endl;
}
```

### Associating a task with a future

```
template<typename Func>
std::future<void> post_task_for_gui_thread(Func f)
{
    std::packaged_task<void()> task(f);
    std::future<void> res=task.get_future();
    std::lock_guard<std::mutex> lk(m);
    tasks.push_back(std::move(task));
	return res;
}
```

### Making (std::)promises

### Saving an exception for the future

 extern std::promise<double> some_promise; try { 	some_promise.set_value(calculate_value()); } catch(...) { 	some_promise.set_exception(std::current_exception());
}

### _**Waiting from multiple threads**_

代码解释

代码改写

std::promise<std::string> p; b Implicit transfer
std::shared_future<std::string> sf(p.get_future());

## _**Waiting with a time limit**_

### time

// clock
std::chrono::system_clock
std::chrono::steady_clock
std::chrono::high_resolution_clock
// duration
std::chrono::duration<short,std:: ratio<60,1>> minutes
std::chrono::duration<double,std::ratio <1,1000>> millisecond
using namespace std::chrono_literals;
auto one_day=24h;
auto half_an_hour=30min;
auto max_time_between_messages=30ms;
// timepoint
std::chrono::time_point<std:: chrono::system_clock, std::chrono::minutes>

### _**Functions that accept timeouts**_

![](https://iwiki.woa.com/tencent/api/attachments/s3/url?attachmentid=18900327)

![](https://iwiki.woa.com/tencent/api/attachments/s3/url?attachmentid=18900328)

## 其他的线程同步方法

### 使用futures进行函数式编程

### 通过消息传递同步操作：Actor模型

思想很简单：如果没有共享数据，每个线程都可以完全独立地进行推理，完全基于它如何响应接收到的消息的行为。因此，每个线程实际上都是一个状态机：当它接收到一条消息时，它以某种方式更新其状态，并可能向其他线程发送一条或多条消息，处理过程取决于初始状态。

### c++ 扩展里的一些同步方法  
  

代码解释

代码改写

// continuations of future
std::experimental::future<int> find_the_answer;
auto fut=find_the_answer();
auto fut2=fut.then(find_the_question);
assert(!fut.valid());
assert(fut2.valid());
// Waiting for more than one future
std::experimental::when_all(results.begin(),results.end()).then(DoneCheck())
// Waiting for the first future in a set with when_any
std::experimental::when_any(results.begin(), results.end()).then(DoneCheck());
// latch
void foo(){
    unsigned const thread_count=...;    latch done(thread_count);    my_data data[thread_count];    std::vector<std::future<void> > threads;    for(unsigned i=0;i<thread_count;++i)        threads.push_back(std::async(std::launch::async,[&,i]{            data[i]=make_data(i);        done.count_down();        do_more_stuff();    }));
    done.wait();    process_data(data,thread_count);
}
// barrier
std::experimental::barrier
std::experimental::flex_barrier
result_chunk process(data_chunk);
std::vector<data_chunk>
divide_into_chunks(data_block data, unsigned num_threads);
void process_data(data_source &source, data_sink &sink) {
    unsigned const concurrency = std::thread::hardware_concurrency();    unsigned const num_threads = (concurrency > 0) ? concurrency : 2;    std::experimental::barrier sync(num_threads);    std::vector<joining_thread> threads(num_threads);    std::vector<data_chunk> chunks;    result_block result;    for (unsigned i = 0; i < num_threads; ++i) {        threads[i] = joining_thread([&, i] {      while (!source.done()) {          if (!i) {            data_block current_block =            source.get_next_data_block();            chunks = divide_into_chunks(            current_block, num_threads);           }           sync.arrive_and_wait();           result.set_chunk(i, num_threads, process(chunks[i]));           sync.arrive_and_wait();           if (!i) {               sink.write_data(std::move(result));           }        }      });  	}
}