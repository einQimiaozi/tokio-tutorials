# 宫崎的流水素面

第2章的结尾我挖了个坑，关于如何在spawn之间安全的传递数据这点我将在本章详细说明，在开始前，请牢记我之前说过的一句话

### 不要通过共享内存来通信，要通过通信来共享内存

这也是tokio中共享内存的精髓

## channel

其实本章主要就是围绕channel展开的，channel是一种能够脱离spawn独立存在的数据传输通道，如果你之前接触过golang，那么其实tokio中的channel和golang中的channel非常相似。

值得一提的是，channel在rust原生异步方案中也提供支持，并且tokio中的channel和rust原生异步方案的使用是一样的！！！看来tokio还真是非常“方便”。

### 一对多channel

channel的默认模式是一对多，即一个接收者和多个发送者。

channel可以附带缓存信息，即：channel中可以同时存多少数据。

当channel中没有数据时，接收者对channel进行接收会引起接收者所在的spawn阻塞。

当发送者发送数据时，如果channel已满，则会引起发送者所在的spawn阻塞。

下面是一个简单的例子

```rust
    use tokio::spawn;
    use tokio::sync::mpsc::channel;

    // 创建一个缓存大小为8的channel
    let (tx, mut rx) = channel(8);
    // 开启一个spawn向channel中发送数据
    tokio::spawn(async move {
        for i in 0..10 {
            tx.send(i).await;
        }
    });
    // 在主线程接收数据并打印
    while let Some(num) = rx.recv().await {
        println!("{}", num);
    }
```


channel在创建时会将发送者和接收者分离，我们需要用两个变量接收channel的返回值。

channel在创建时一般不需要指定channel中存储数据的类型，在第一次调用tx或rx时，rust编译器会自动判断，当然你也可以显示指定。

```rust
    use tokio::sync::mpsc::{Sender,Receiver};
    
    let (tx, mut rx):(Sender<i32>,Receiver<i32>) = channel(8);
```

哎？不是说一对多么，但是在上面的代码中我们只看到了“一对一”啊？

别急，通过对Sender进行拷贝，是可以做到一对多的，Sender实现了Copy trait，并且这个Copy并不是我们理解中的“拷贝副本”，而是针对一份Sender的多个合法引用。

```rust
    let (tx, mut rx):(Sender<i32>,Receiver<i32>) = channel(8);
    let tx2 = tx.clone();
    tokio::spawn(async move {
        for i in 0..10 {
            tx.send(i).await;
        }
    });
    tokio::spawn(async move {
        for i in 10..20 {
            tx2.send(i).await;
        }
    });
    while let Some(num) = rx.recv().await {
        println!("{}", num);
    }
```

rx可以正常的接收到tx和tx2发送的数据，但是顺序并不保证。

既然Sender可以Copy，那么Receiver是不是也可以通过Copy直接实现多对多啊？确实，这样的思路是可以实现多对多的，但是很遗憾，Receiver并没有实现Copy trait，所以我们并不能copy rx。

```rust
24 |     let rx2 = rx.clone();
   |                  ^^^^^ help: there is an associated function with a similar name: `close`
```

道理也很简单，Sender允许实现Copy trait是因为发送时不会产生多个Sender同时需要访问同一个临界资源的问题，而Receiver在接收时则不同，如果两个Receiver在不通的spawn中同时获取到了一份临界资源，那么就会产生并发安全问题。

### 一对一channel

除了一对多channel，rust和tokio还提供了一种一对一channel，oneshot。

其实在上面介绍一对多channel时，第一个案例就是一个一对一channel的实现，那么为什么tokio还要提供一对一channel呢？

其实主要就是性能，tokio中提供的一对一channel性能要比一对多channel更好，更加适合一些只需要一对一传递数据的场景。

```rust
    use tokio::sync::oneshot;

    let (tx, mut rx) = oneshot::channel();
```

oneshot的使用方法和正常的channel一样，需要注意的就是，既然是一对一channel，那么Sender必然也是无法Copy的。

关于oneshot的典型应用场景将在后续章节内介绍，这里可以先简单提一下，oneshot经常用于被一对多channel作为数据传递，之后作为两个spawn之间的“私密”通信信道使用。

### 多对多channel

最后的channel是多对多channel，名为broadcast。

broadcast可以通过多个Sender和多个Receiver实现多对多的channel。

```rust
use tokio::sync::broadcast;

let (tx,mut rx) = broadcast::channel(16);
    let mut rx2 = tx.subscribe();
    tokio::spawn(async move {
        loop {
            if let Ok(num) = rx.recv().await {
                println!("{}", num);
            }
        }
    });
    tokio::spawn(async move {
        loop {
            if let Ok(num) = rx2.recv().await {
                println!("{}", num);
            }
        }
    });
    for i in 0..8 {
        tx.send(i).unwrap();
    }
```

在主线程中send的数据会被两个spawn获得，注意，这里会有临界资源访问安全问题，即

```rust
tx.send(1);

let num1 = rx.recv().unwrap();
let num2 = rx2.recv().unwrap();

// 此时num1和num2指向同一份数据
```

多对多channel的应用场景不是特别多，最常见的应用场景就是广场式聊天室和上古时代的bbs。

## 本章作业

 构建一个私密通信系统，由主线程发送消息，每次接收消息创建一个spawn，同时该spawn使用一个专用信道和主线程通信，实现echo主线程的消息
 
 ```rust
 use tokio::sync::mpsc;

struct Message {
    // your code
}

#[tokio::main]
async fn main() {
    let names = vec!["name1","name2","name3"];
    let (tx,rx) = mpsc::channel(8);
    // your code
    tokio::spawn(async move {
        for i in 0..3 {
            
        }
    });
}
 ```


## 上一章作业答案

### 1

```rust
 #[tokio::main]
async fn main() {
    let t1: JoinHandle<&str> = tokio::spawn(async {
        println!("tokio");
    });
    let t2: JoinHandle<&str> = tokio::spawn(async {
        println!("edo");
    });
    let t3: JoinHandle<&str> = tokio::spawn(async {
        println!("hello");
    });
    t3.await.unwrap();
    t1.await.unwrap();
    t3.await.unwrap();
    t2.await.unwrap();
}
```

### 2

```rust
#[tokio::main]
async fn main() {
    let t1: JoinHandle<&str> = tokio::spawn(async {
        return "str";
    });
    let res = t1.await.unwrap();
    tokio::spawn(async move {
        println!("{}",res);
    }).await;
}
```
