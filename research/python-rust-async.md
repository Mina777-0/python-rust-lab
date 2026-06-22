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

    Python asyncio Component                Tokio (Rust) Equivalent                 Description

    asyncio.create_task(coro)               tokio::spawn(future)                    Spawns a concurrent background task on the runtime.

    loop.create_future()                    tokio::sync::oneshot::channel()         Creates a one-time producer/consumer channel acting as a Future.

    loop.add_reader(fd, callback)           tokio::io::unix::AsyncFd                Registers a raw file descriptor to monitor read readiness via Epoll/Kqueue.

    asyncio.gather(*aws)                    tokio::join! or                         Waits for multiple asynchronous operations to complete concurrently.
                                            futures::future::join_all   

    asyncio.sleep(delay)                    tokio::time::sleep(duration)            Pauses execution asynchronously without blocking the OS thread.

    asyncio.Queue                           tokio::sync::mpsc                       Provides an asynchronous multi-producer, single-consumer channel.


# References:

    https://medium.com/@OlegKubrakov/practical-guide-to-async-rust-and-tokio-99e818c11965
    https://dev.to/_56d7718cea8fe00ec1610/understanding-async-socket-handling-in-rust-from-tcp-request-to-waker-wake-up-19le
    https://doc.rust-lang.org/book/ch17-01-futures-and-syntax.html
    https://tokio.rs/tokio/tutorial/spawning
