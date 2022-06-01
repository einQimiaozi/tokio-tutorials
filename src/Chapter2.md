# 前有江户

在上一章中我们介绍了tokio启动一个并发任务的基本用法，现在我们来看下面一个例子

```rust
let edo = "tokio";
tokio::spawn(async {
    println!("{}",edo);
});
```

尝试运行上面的代码，会造成编译错误

```
error[E0373]: async block may outlive the current function, but it borrows `edo`, which is owned by the current function
 --> src/main.rs:4:24
  |
4 |       tokio::spawn(async {
  |  ________________________^
5 | |         println!("{}",edo);
  | |                       --- `edo` is borrowed here
6 | |     });
  | |_____^ may outlive borrowed value `edo`
  |
  = note: async blocks are not executed immediately and must either take a reference or ownership of outside variables they use
help: to force the async block to take ownership of `edo` (and any other referenced variables), use the `move` keyword
  |
4 |     tokio::spawn(async move {
  |                        ++++

For more information about this error, try `rustc --explain E0373`.
error: could not compile `untitled2` due to previous error
```

咦？看来我们不能让江户城变为东京了

不过通过错误信息，我们可以发现，编译器让我们使用move关键字，哈哈哈，如果你之前曾经使用过golang的闭包，那么你应该对这样的错误信息很眼熟。

## move

实际上每一个spawn相当于执行了一个匿名方法，所以必然的，我们需要转移该方法内使用的外部变量所有权。

而且转移所有权是必须的，因为编译器无法得知，spawn和其外部代码哪个先执行完毕（即便你使用了await关键字，如果你的spawn内使用了如循环等操作，那么依然会让编译器造成这样的困扰）。

我们根据编译器提示修改下面的代码即可。

```rust
let edo = "tokio";
tokio::spawn(async move{
    println!("{}",edo);
});
```

江户城终于变成东京都了，但是这样的做法还是有问题，如果我们希望使用多个spawn同时输出一个vec中的内容

```rust
let name = vec!["tokio","edo"];
for i in 0..10 {
    tokio::spawn(async move {
        for i in &name {
            println!("{:?}",name);
        }
    }).await;
}
```

哈哈哈，又报错了，我们来看看这次的错误是什么

```
error[E0382]: use of moved value: `name`
  --> src/main.rs:6:33
   |
3  |       let name = vec!["tokio","edo"];
   |           ---- move occurs because `name` has type `Vec<&str>`, which does not implement the `Copy` trait
...
6  |           tokio::spawn(async move {
   |  _________________________________^
7  | |             for i in &name {
   | |                       ---- use occurs due to use in generator
8  | |                 println!("{:?}",name);
9  | |             }
10 | |         }).await;
   | |_________^ value moved here, in previous iteration of loop

```

编译器已经说的很清楚了，我们在spawn内部发生了所有权转移。

实际上async move这样的用法，会将外部变量的所有权移动到spawn内部，如果在外部再次尝试获取外部变量，则当前外部变量的所有权已经不存在了。

所以实际上我们只需要将一份外部变量的拷贝交给spawn即可，这样外部方法继续持有变量的所有权，还可以再次交给下一个spawn。

```rust
let name = vec!["tokio","edo"];
for i in 0..10 {
    // 在这里拷贝name
    let name = name.clone();
    tokio::spawn(async move {
        for i in &name {
            println!("{:?}",name);
        }
    }).await;
}
```

大功告成，通过这个例子也引入了一个新问题，我们到底能够在spawn中安全的获取什么样的外部变量？

答案其实很简单，实现了Copy或Send trait的变量，或者一个生命周期为static的变量，总之就是你能保证，spawn得到的变量所有权不会再spawn执行期间失效即可。

前面的例子就是讲一个实现了Copy trait的变量移入spawn中，关于Send trait，指的是实现该trait的对象都可以在spawn中安全的发送。

## Arc

假设我定义了一个对象，但是我不希望该对象实现Copy或者Send trait，或者我使用的对象并没有实现Copy或Send trait，但是我依然希望在不用的spawn中共享它。

上面的想法确实值得思考，但是首先我把结论放在这里，这样的想法是很 “危险” 的。

在不同的spawn中共享同一个对象，有可能引发内存安全问题。

假设我们将对象a在spawn1和spawn2中共享，spawn1转移了对象a中某个成员的所有权，对于spawn2，你将会获取到一个未知的吧啦吧啦吧啦不知道是什么的东西，你看，是不是非常可怕！！！

但是我就是比较杠，我就是要这么做，ok，回忆一下学习rust时我们希望共享一个对象的方法。

使用Rc或Arc。

Rc和Arc使用的区别其实仅仅是能够在并发环境下使用，Rc面向非并发环境，可以通过Rc将对象在不同方法之间共享，Arc则面向并发环境，关于Rc和Arc的实现原理我不多做阐述，如果你有兴趣可以参考Rust官方教程或孙飞老师的rust语言圣经。在这里我们只需要知道不论是Rc还是Arc，它们都会通过一个计数器，对当前对象被共享的次数进行计数，当计数为0时，该对象才会根据所有权模型进行释放。如果你对jvm比较熟悉，其实这种做法就是一个手动版管理内存的引用计数法。

```rust
use std::sync::Arc;

struct Object {
    name: String,
}
```

```rust
// 通过Arc对name进行共享
let obj = Arc::new(Object{name: "tokio".to_string() });
for i in 0..10 {
    let obj = obj.clone();
    // 在这里拷贝name
    //let name = name.clone();
    tokio::spawn(async move {
        println!("{:?}",obj.name);
    }).await;
}
```

问题解决了，我们可以在不同spawn共享我们的obj，但是要注意，如果你希望修改任何值，还需要搭配Cell或RefCell，但即便如此，rust也不能保证你一定能够通过共享对象实现你想要的功能，比如在web应用中，你可以通过共享Socket套接字来进行读写分离，但是实际上在rust中一个被Arc共享的Socket套接字是无法调用Read和Write方法的，这样的做法并不是一个好的方案！！

那么如何能够安全的共享对象呢？这里引用golang中的一个设计思路：

### 不要通过共享内存来通信，要通过通信来共享内存

具体的做法我将在第4章中说明，目前你只需要记住，使用spawn时不到万不得已，不要尝试通过Arc共享内存，但是你确实可以这样做。

## 本章作业

### 不使用Arc实现本章中在不同spawn中输出obj.name

提示：spawn无法共享一个没有实现Copy trait的对象，但是可以共享实现了Copy trait的对象

## 上一章作业答案

1. 代码会进入死循环，因为对一个spawn调用await会阻塞当前线程，直到该spawn执行结束，代码中的spawn执行的是一个死循环，所以main线程永远不会结束

2. 本题只需要使用一个异步方法即可，spawn不使用await进行阻塞，只对异步方法进行阻塞，这样可以真正实现两个spawn并发执行

```rust
#[tokio::main]
async fn main() {
    loop {
        tokio::spawn(async {
            hello("tokio").await
        });
        tokio::spawn(async {
            hello("tokyo").await
        });
    }
}

async fn hello(name: &str) {
    println!("{}",name);
}
```
