# alone in tokio

tokio是一个第三方rust异步编程框架，也是目前rust中采用的主流并发解决方案框架。

rust中虽然原生支持异步编程，提供了async/await等关键字，但是相比tokio，使用起来更为复杂，如果你希望创建一个高可用并且鲁棒性强的异步web服务器，那么tokio将是你最好的选择。

基于rust强大的性能，tokio在并发性能上也远好于java/golang的并发解决方案。

在阅读本手册前，请确保您已经基本掌握rust语言，当然我不会要求您是一个rust高手，像lifetime这样的概念确实难以理解和使用，实际上大部分时间您也不需要在项目中真正涉及大量的手动标注lifetime，只需要您知道，rust如何编写，理解所有权模型，借用等概念，如果你能够掌握vector，hashmap等集合那么更好。

好了，多说无益，下面让我们正式开始迷失东京吧。

## async/await

在开始前我们需要在项目中导入tokio框架，毕竟前面说过了，tokio是一个第三方框架。

在cargo.toml中导入tokio

```
tokio = {version = "1.17.0", features = ["full"]}
```

本教程编写时tokio的最新稳定版为1.17.0，如果后续tokio发布新版本，也欢迎您尝试使用最新版本的tokio学习。

async和await是rust异步编程中最重要的两个关键字，实际上rust原生异步编程也主要依靠它们，让我们先创建一个异步方法吧！！

首先我们将main指定为一个tokio异步方法

```rust
#[tokio::main]
async fn main() {
  do_something......
}
```

被async关键字修饰的方法即为一个异步方法，任何异步调用都需要在async方法内执行

接下来我们开启一个spawn，你可以理解为，一个spawn就是一个rust协程，所以我们不必关心你当前的硬件状态到底有多少cpu，支持多少核心线程

```rust
tokio::spawn(async {
        do_something.....
    })
```

同样，一个spawn内部执行的也是一个异步方法，我们可以通过隐式调用创建一个异步方法，当然你也可以直接编写一个异步方法然后显示调用它

```rust
    tokio::spawn(async {
        hello_tokio();
    });
```

现在，让我运行我们的第一个异步程序吧！！！

```rust
#[tokio::main]
async fn main() {
    tokio::spawn(async {
        hello_tokio();
    });
}

async fn hello_tokio() {
    println!("hello tokio");
}
```

唉？为什么没有输出？？？

不要紧张，放松，世界没有崩溃，我们现在我们的异步程序没有输出是正常的，我们只需要在我们调用的异步方法后面加入 await 关键字就可以了！！

```rust
#[tokio::main]
async fn main() {
    tokio::spawn(async {
        hello_tokio().await;
    });
}

async fn hello_tokio() {
    println!("hello tokio");
}
```

现在我们的程序就可以输出 hello tokio 啦！！！

### 要理解的事情

当我们使用async创建一个异步方法时，该方法被调用后会异步的执行，即：hello tokio会和main同时执行，也就是说main不会等待hello tokio执行结束，所以在hello tokio执行结束前，main就已经执行结束了，当然，hello tokio也不会有任何输出了！！！

使用await关键字，可以让当前调用异步方法的线程等待该方法执行结束，具体的说，当前线程会阻塞在方法调用这一行代码处，等待异步方法执行结束后，再次从改行代码开始向下继续执行

```
1   #[tokio::main]
2   async fn main() {
3       tokio::spawn(async {
4           // main方法会在此阻塞，当hello tokio执行结束后会继续切换到代码的第5行开始执行
5           hello_tokio().await;
6       });
7   }
8 
9   async fn hello_tokio() {
10       println!("hello tokio");
11   }
```

如果你学习过golang，其实你会发现，await关键字和golang中的waitgroup很像，唯一的区别是await会阻塞在使用该关键字的那一行，而waitgroup只是单纯的使用统计计数方案统计当前线程下还有多少没有执行完毕的协程。

await关键字可以在任何并发方法或spawn上调用，我们可以将上面的代码改写为下面的方式，输出结果是一样的

```rust
#[tokio::main]
async fn main() {
    tokio::spawn(async {
        println!("hello tokio");
    }).await;
}
```

好了！现在你已经学会通过async和await如何创建一个异步方法了！本章到此结束，当然你的工作还没有结束，我会在每章后面留下一个简单的课后作业，我希望你能够完成它，如果你仔细阅读了本章的内容，那么作业对于你来说应该十分简单。

## 本章作业

### 1. 以下代码会输出什么？

请尝试推断出以下代码会输出什么结果

```rust
#[tokio::main]
async fn main() {
    tokio::spawn(async {
        hello_tokio().await;
        println!("hello tokio");
    });
}

async fn hello_tokio() {
    loop {

    }
}
```

在回答前，请尽可能自己思考，如果你实在想不出，可以将上面的代码复制到你的编译器中进行编译，查看结果。

### 2.编写一个程序，让两个spawn同时输出 hello tokio 和hello tokyo，输出顺序没有限制
