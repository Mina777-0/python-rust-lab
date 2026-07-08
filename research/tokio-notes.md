# How Does Tokio Work

* Tokio runs on top of your computer's operating system threads. When you start a standard Tokio application, it launches a multi-threaded work-stealing runtime
The Main Thread: This is the primary thread running your async fn main(). It kicks off the application.Background Threads (Worker Threads): Tokio automatically 
spawns a pool of extra threads in the background (usually equal to the number of CPU cores your computer has).When you call tokio::spawn, you are taking a task 
off the main thread and handing it to Tokio's scheduler. Tokio then assigns that task to one of these background worker threads to execute independently. 
This keeps your main thread free to handle other incoming events.

# Tokio::spawn 

* tokio::spawn() is used to schedule tasks (Future trait) to run at the background. It returns JoinHandle<>. 

        pub fn spawn<F>(future: F) -> JoinHandle<F::Output> 
            where
            F: Future + Send + 'static,
            F::Output: Send + 'static, 


- JoinHandle<T> where T here is the return value type of the spawned task.
- JoinHandle<()> the spawned task doesn't return a value.

There are two ways to spawn tasks.
1. tokio::spawn() 
2. tokio:spwan(async {})

- tokio::spawn(), the task evaluates immediately on the current thread, and whatever it returns gets sent to the background thread. 
Spawn only accpets futures, so the task() has to be future (async) or returns a type that implement Future trait. 
If the task is sync and returns non-Future trait, it will crash during the compilation. 

- tokio::spawn(async {}) defers evaluation. async block creates a lazy Future. task() will execute later, inside the Tokio background worker pool.
The good thing here is that task() can be sync or async. async block will wrap it as a Future and execute it. 
If the task is async future, async block will wrap the Future inside a Future, so we need to .await the outside Future wrapper
Future<output= Future<...>>


        // This future tasks
        async fn future_task(id:i32) -> String {
            let s= format!("This is task with id: {}", id);
            s
        }


        // tokio::spawn()
        #[tokio::main]
        async fn main() {
            let mut tasks= Vec::new();
            for i in 0..3 {
                tasks.push(tokio::spawn({task(i)}));
            }

            let mut outputs= Vec::new();
            for task in tasks {
                outputs.push(task.await.unwrap())
            }
            
            println!("Outputs: {:?}", outputs);
        }



        #[tokio::main]
        async fn main() {
            let mut tasks= Vec::new();
            for i in 0..3 {
                // move since we hands async block the ownership of i
                tasks.push(tokio::spawn(async move {task(i).await}));
            }

            let mut outputs= Vec::new();
            for task in tasks {
                outputs.push(task.await.unwrap())
            }
            
            println!("Outputs: {:?}", outputs);
        }



# How to await more than one spawned task concurrently?

* tokio::join!()
* futures::future::join_all()
* tokio::task::JoinSet


* tokio::join!() accepts only the Future. If the spawned tasks are stores in a Vec, join!() will fail. It doesn't accept Vec<Future>

        #[tokio::main]
        fn main() {
            let handle1= tokio::spawn(async {
                // async function
            });

            let handle2= tokio::spawn(async {
                // async function
            });

            let handle3= tokio::spawn(async {
                // async function
            });

            tokio::join!(handle1, handle2, handle3);

        }

* futures::future::join_all() accepts Vec<Future>


        #[tokio::main]
        async fn main() {
            let mut tasks= Vec::new();
            for i in 0..3 {
                tasks.push(tokio::spawn(async move {task(i).await}));
            }

            let results= futures::future::join_all(tasks).await; // returns Result<T, JoinError> 
            for res in results {
                if let Ok(_result) = res {
                    println!("Got: {}", _result);
                }
            }
        }


* tokio::task::JoinSet 

        #[tokio::main]
        async fn main() {
            let mut set= JoinSet::new();
            
            for i in 0..10 {
                set.spawn(async move { 
                    task(i).await;
                });
            }
            
            while let Some(res) = set.join_next().await{ // returns Option<Result<T, JoinError>>>
                match res {
                    Ok(string_res) => println!("Got: {:?}", string_res),
                    Err(e) => eprintln!("Task panicked or cancelled: {}", e),
                }
            }
        }
        