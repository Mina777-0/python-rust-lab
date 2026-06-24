# Asynchronous philosophy between Python asyncio and Rust Tokio

In Python, asynchronous functions are called Coroutines. The equivalent semantic in Rust is Future. Future in Rust is a trait which has method poll()

    trait Future {
        type Output;
        fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
    }

    pub enum Poll<T> {
        Ready(T),
        Pending,
    }


# Runtime 

A program must call poll on a Future to make progress. A Future will either return Ready(value) or Pending and schedule itself to be polled again. 
The program can do this in a loop or use a Waker to be notified when the resource the Future is waiting on is ready.

In Python asyncio is the event loop manager, while in Rust Tokio is the event loop manager. The difference between asyncio and Tokio is that asyncio works on
a single thread, while Tokio uses a multi-threaded, work-stealing thread pool. Work-stealing means threads will pull tasks from other threads if they don’t 
have anything to do.

    async def main():
        pass 

    if __name__ == "__main__":
        asyncio.run(main())

    #[tokio::main]
    async fn main {
        todo!();
    }

# Scheduling 

    # Python
    async def demo():
        print("Hello World")

    async def main():
        task= asyncio.create_task(demo())
        await task 

    if __name__ == "__main__":
        asyncio.run(main())


    # Rust

    pub async fn demo() {
        println!("Hello from async runtime Tokio");
    }


    #[tokio::main]
    async fn main() {
        // tokio::spawn == asyncio.create_task()
        let handler= tokio::spawn(async_demo::demo());

        handler.await.unwrap();
    }


# Blocking event loop

What happens when hit a while loop?
Since asyncio event-loop works on a single thread, it freezes completely the entire application once it hits whiel True unless there is an await, it yields 
the control back to the event-loop. The event loop will be blocked forever 

In Rust, it does freezes once it hits loop, but since it uses multi-threaded, work-stealing, it freezes only one Thread unless there is an await. Tokio will 
automatically move (steal) your other asynchronous tasks to different threads so the app doesn't crash
If you have a heavy loop in Rust and want to be a "good citizen" without hitting a real I/O pool, you must explicitly yield control back to Tokio 
using tokio::task::yield_now().await




# Mapping asyncio to Tokio

| Python asyncio Component | Tokio (Rust) Equivalent | Description
| --- | --- | --- |
| asyncio.create_task(coro) | tokio::spawn(future) | Spawns a concurrent background task on the runtime |
| loop.create_future() | tokio::sync::oneshot::channel() | Creates a one-time producer/consumer channel acting as a Future |
| loop.add_reader(fd, callback) | tokio::io::unix::AsyncFd | Registers a raw file descriptor to monitor read readiness via Epoll/Kqueue |
| asyncio.gather(*aws) | tokio::join! or | Waits for multiple asynchronous operations to complete concurrently |
| | futures::future::join_all | | 
| asyncio.sleep(delay) | tokio::time::sleep(duration) | Pauses execution asynchronously without blocking the OS thread |
| asyncio.Queue | tokio::sync::mpsc | Provides an asynchronous multi-producer, single-consumer channel |



# Async TCP socket 

* Asyncio:
It all happens at the OS level. Once we bind or create a listener on a port (i.e. 8080), the OS listens on this port for any TCP handhsakes and later packages   to send to the corresponding process. 

        async def start_serer():
            ser_scoket= socket.socket(family=socket.AF_INET, type= socket.SOCK_STREAM)
            ser_scoket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEPORT, 1)
            ser_scoket.setblocking(False)
            ser_scoket.bind(("127.0.0.1", 8080))
            ser_scoket.listen(1)
    
            loop= asyncio.get_running_loop()

            # Accepting connection is handled by the event-loop.
            cl_sock, addr= await loop.sock_accept(sock)

  
  Here the OS kernel creates a new socket with FD (unique file descriptor identifier). The listening socket is non-blocking, the application can poll it or ask the OS to             notify it when the connection is ready. On the other hand event-loop creates a Future object and invokes a reader
          
            # loop.add_reader(fd, callback) 
            # this reader takes the fd of the listening socket and add the Future object as callback 
            waker= loop.ceate_future()
            fd= ser_sock.fileno()
            loop.add_reader(fd, lambda: waker.set_result(None) or waker.done())
            try:
                await waiter 
            finally:
                loop.remove_reader(fd)

- Event loop puts the coroutine to sleep. The socket remains registered on the OS Kernel's interest list.
- Event loop goes to handle other tasks in the queue
- When the OS socket recieves is ready, it notify the event-loop. It put the Future obj to be executed and marked as done().
- It accpets the handshake and puts it in the queue to be handled 


* Tokio:
  
        async fn start_server() -> Result<(), io::Error>{
            let listener= TcpListener::bind("127.0.0.1:8080").await?;
            let (cl_sock, addr)= listener.accept().await?;
        }

* The listener or the cl_sock are both non-blocking because Tokio uses O_NONBLOCK flag on fd.
Here Tokio create a Future but a Future trait

        trait Future {
            type Output;
            fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
        }

        pub enum Poll<T> {
            Ready(T),
            Pending,
        }

Tokio calls .poll() on this future
Returns Poll::Pending
Calls Reactor::register_fd() to register the socket FD with epoll
Stores the Waker to be notified later
The task is paused (not executed again until woken up)

Later:
The Reactor calls epoll_wait() in a loop
When the socket becomes readable, the OS notifies the reactor
The stored Waker is called → waker.wake()
The task is placed back in the async runtime’s queue and is polled again
This time, .poll() returns Poll::Ready and handshake is done.


| asyncio | Tokio | Discription |
| --- | --- | --- |
| .add_reader(fd, cb) | Reactor::register_fd(fd, waker) | register the FD with epoll/Kqueue so the runtime can safely yield execution |
| future.set_result(None) | waker.wake() | Invoke OS epoll_wait() detect readiness. inform the executor to schedue the task |
| remove_reader(fd) | Drop | clear the FD from the monitoring list |
| future.done() == True | Poll::Ready(Result) | Signal the executioner that the process is completely processed |





# References:

    https://medium.com/@OlegKubrakov/practical-guide-to-async-rust-and-tokio-99e818c11965
    https://dev.to/_56d7718cea8fe00ec1610/understanding-async-socket-handling-in-rust-from-tcp-request-to-waker-wake-up-19le
    https://doc.rust-lang.org/book/ch17-01-futures-and-syntax.html
    https://tokio.rs/tokio/tutorial/spawning
    https://dev.to/_56d7718cea8fe00ec1610/understanding-async-socket-handling-in-rust-from-tcp-request-to-waker-wake-up-19le
