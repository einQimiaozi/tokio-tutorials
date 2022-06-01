# 风仁无幻

本章是本书中为数不多的理论较多的篇章，其实在写之前我也考虑过要不要写这一部分，毕竟本书的主题是 “简明教程”，即我希望更加通俗易懂的让大家通过阅读本书快速入门tokio。

但是想来想去，还是专门写一章简单讲一下tokio中比较重要的理论部分吧，毕竟亚里士多德说过：Human knowledge has three：theory，practical，identification。

## async到底是什么

如果说前两章中出现最多的tokio关键字是什么，那一定是async了，所有的异步方法和对异步方法的调用，都离不开async。

asynchronous，async的全称，在rust中，如果要明确的说，async可以更准确的认为是async future。

那么就有观众要问了，future又是啥啊，看起来tokio中又出现了一个麻烦的东西。

其实future是rust原生并发中的一个概念，而tokio则是基于原生并发的封装（更准确的说是基于mio+原生并发的封装，实际上mio就是tokio的前身）。

简单来说，rust使用了一种名为async future的并发处理方案，该方案使用一种类似状态机的手段以近乎零开销的代价处理并发操作，future最早由推特工程师团队提出，关于future的更多详细解释可以参考它的原始论文

https://monkey.org/~marius/funsrv.pdf

简单来说，future通过一个callback方法，不断监听一个poll，当poll返回ready状态时，代表当前的poll对某个异步操作执行完毕，而返回Pending时，则说明异步任务还未执行完毕，poll会在下次执行完毕后再次调用callback方法通知future。

关于poll的概念我不再此多做阐述，如果你了解linux或者一些异步网络编程，那么你应该对select，poll，epoll这些事件驱动方案有一定了解，如果你还不了解，那么请停止阅读，去了解一下吧，这部分内容对于异步编程来说即基础，又重要。

实际上在使用rust原生并发操作时，我们调用await后返回的对象，就是一个future，future就是一个用于执行一步操作的容器，下图是对future执行流程的一个大致说明

![future](https://github.com/einQimiaozi/tokio-tutorials/blob/main/img/future.png)

通过图我们来一点一点解剖future吧！

首先future会不断接受一个异步任务，并将其派发到一个线程队列中。

后台有一个专门的线程通过poll监听任务队列，通过该线程将任务派发给event loop，event loop就是一个thread pool实现的线程组。

event loop给线程分配任务，通过poll对具体任务监听，poll后会有两类状态（即图中的5），Ready或Pending。

当Ready时，直接将结果推回给future，future通过返回值将最终结果返回给外部调用await的方法。

当Pending时，说明future还有任务还没有执行完毕，此时该任务会继续在原地等待（即通过调用await阻塞），并且future继续执行，此时wake方法将被第一次使用。

wake是一个callback方法，将wake注册给异步操作，使得wake能够通知当前异步操作，多说一句，在源码中使用了一个上下文Context传递并调用wake。

当future任务执行完毕后，会通过wake方法通知event loop，event loop将执行结果和任务重新推回给任务队列，并将poll状态置为Ready

future此时再次访问任务队列时，监听到Ready状态，会将该任务的返回值推回给调用await的方法，执行完毕。

啊，是不是有点糊涂了，不过这部分确实也是本书中最难以理解的部分了，如果第一次没有理解，可以搭配上面的流程图多读几次，下面我会给出一个future的伪代码简单实现，帮助你更加简单的理解future。

```rust
struct service<'a> {
  stream: &'a TcpStream
}

impl Future for service<'_> {
    type Output = Vec<u8>;

    fn poll(&mut self, wake:fn(), cx: &mut Context<'_>)
        -> Poll<Self::Output>
    {
        // 如果当前的socket流有消息
        if self.stream.recv() {
          // 读取消息并通过Ready返回给future
          Poll::Ready(self.stream.read())
        } else {
          // 否则将wake通过context注册给socket流
          self.socket.set_callback(cx.wake());
          Poll::Pending
        }
    }
}
```

我们假设了一个读取socket流数据的服务，通过poll不断的给future传递消息并监听任务状态

### 理解future能够帮助你更好的理解tokio，但如果暂时无法理解future，也不需要花费太多时间死磕，本书的目的仍然是能够让你快速上手使用tokio，实际上我在使用tokio写出第一个异步web服务时，并不理解future，所以暂时没有理解也请不要担心，当你熟练的使用tokio后再回来看这部分即可。
