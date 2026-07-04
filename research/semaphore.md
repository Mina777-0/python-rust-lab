# Sempahore 

* Have you ever faced a need to handle the traffic for thousands concurrent operations? API limits, database connections, or external services. Semaphore is the magic module that works as a traffic light to control the busy traffic of concurrent operations. 

* Here I will show you how to use semaphore in Python and Rust using simple examples and then show how to use them to control sockets traffic. 

# Python

* Here we try to create 10 concurrent operations at the same time. We detrmine that the traffic to be 3; Semaphore(3). Before we create the operation, we acquire the semaphore, then create the operation. This way we semaphore will make sure that only the allowed number of operations are created and added to the list of tasks.
Once the first task is run, then the semaphore will allow only two more, and then stop. This way will allow only three tasks to be done, while the code will keep waiting. We need to realease the sempahore to allow the rest of the tasks to pass. The position of the release determine how the tasks will be handled.


async def sem_demo():
    tasks= []
    semaphore= asyncio.Semaphore(3)

    for i in range(1,10):
        # Acquire the traffic
        await semaphore.acquire()
        task= asyncio.create_task(handler(i))

        tasks.append(task)
        print(f"Task {i} is a added to the list")

    await asyncio.gather(*tasks)

async def handler(task_id):
    print(f"Task-{task_id} started")
    await asyncio.sleep(2.0)
    print(f"Task-{task_id} finished")
    

* Here we release it once the task is scheduled to the list, this will always create a free slot for the next task to be added even before the first task is finished

async def sem_demo():
    tasks= []
    semaphore= asyncio.Semaphore(3)

    for i in range(1,10):
        # Acquire the traffic
        await semaphore.acquire()
        task= asyncio.create_task(handler(i))

        tasks.append(task)
        print(f"Task {i} is a added to the list")

        # Release the traffic
        semaphore.release()

    await asyncio.gather(*tasks)

async def handler(task_id):
    print(f"Task-{task_id} started")
    await asyncio.sleep(2.0)
    print(f"Task-{task_id} finished")
    

* This way we pass the semaphore permit to the handler, so once it's finished, the permit is released and create a free slot for the next task to be executed. But this is done only after the task is finished. This way we control the traffic of the conccurent operations 

async def sem_demo():
    tasks= []
    semaphore= asyncio.Semaphore(3)

    for i in range(1,10):
        await semaphore.acquire()
        task= asyncio.create_task(handler(i, semaphore))

        tasks.append(task)
        print(f"Task {i} is a added to the list")

    

    await asyncio.gather(*tasks)
    

async def handler(task_id, permit:asyncio.Semaphore):
    print(f"Task-{task_id} started")
    await asyncio.sleep(2.0)
    print(f"Task-{task_id} finished")
    permit.release()


# Rust 

tokio::Semphore philiosphy is different from asyncio.Semaphore.
In Rust we acquire, but the release happens autmotically at the end of the scope - no need to release or drop.


* Here the tasks are spawned, but only three are executed, then when a new slot appears, another spwaned task is executed.


async fn sem_demo() {
    let semaphore= Arc::new(Semaphore::new(3));
    let mut tasks= Vec::new();

    
    for i in 1..10 {
        let cloned_sem= semaphore.clone();

        // Now we spawn only the permitted tasks
        let task= tokio::spawn(async move {
            let permit= cloned_sem.acquire().await.unwrap(); // PUASES HERE

            if let Err(e) = handler(i).await {
                eprintln!("Handler failed: {}", e);
                return;
            }
            // Permit is released here automatically
        });

        tasks.push(task);
        println!("Task {} is added to the list. List-size= {}", i, tasks.len());

    }

    for task in tasks {
        let _= task.await;
    }
}

async fn handler(task_id: usize) -> Result<(), io::Error>{
    println!("Worker-{} started", task_id);
    tokio::time::sleep(Duration::from_secs(1)).await;
    println!("Worker-{} finished", task_id);

    Ok(())

}


* Here the permit will pause the spawning, but once the first iteration is finished, the permit will drop automatically allowing the next spawned task to start and so on.

async fn sem_demo() {
    let semaphore= Arc::new(Semaphore::new(3));
    let mut tasks= Vec::new();

    
    for i in 1..10 {
        let cloned_sem= semaphore.clone();
        let permit= cloned_sem.acquire_owned().await.unwrap(); // PUASES HERE

        // Now we spawn only the permitted tasks
        let task= tokio::spawn(async move {
            
            //tokio::time::sleep(Duration::from_secs(2)).await;
            if let Err(e) = handler(i).await {
                eprintln!("Handler failed: {}", e);
                return;
            }
            
        });

        // Permit is released here automatically
        
        tasks.push(task);
        println!("Task {} is added to the list. List-size= {}", i, tasks.len());
        //drop(_permit); // Drop the permit after it's finished so more tasks can be permitted 

    }

    for task in tasks {
        let _= task.await;
    }
}

async fn handler(task_id: usize) -> Result<(), io::Error>{
    println!("Worker-{} started", task_id);
    tokio::time::sleep(Duration::from_secs(1)).await;
    println!("Worker-{} finished", task_id);

    Ok(())

}

Sempahore.acquire() returns SemaphorePermit<'a> so its life time is bound to the semaphore that created it.
Semaphore.acquire_owned() return OwnedSemaphorePermit its life time is static.


# References:
https://medium.com/@mr.sourav.raj/mastering-asyncio-semaphores-in-python-a-complete-guide-to-concurrency-control-6b4dd940e10e
