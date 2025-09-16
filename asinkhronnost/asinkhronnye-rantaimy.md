# Асинхронные рантаймы

## Проблемы тред-пулов

To understand the need of a good async runtime we need to look at problems that occur with thread pools in I/O bound applications.

1\) Application uses a fixed-size thread pool to serve incoming requests.

E.g. Spring over Tomcat by default uses 200 threads to serve incoming requests. If all the threads are occupied, newly arrived requests will have to wait in a queue keeping the socket.

2\) Application performs executes a various number of flow (multi-steps functionality consists of computations and I/O) in parallel, and flows can also spin up parallel executions.

After the number of threads grows above the certain threshold (specific to the CPU and OS), the system begins to spend too much time on context switching, which affects the performance dramatically.

## Виды рантаймов

The well knows approaches for building concurrent applications are:

* OS threads – good for compute intensive programs
* Events and callbacks – good for I/O bound applications
* Coroutines – good for application that have a lot of I/O, but also containing computations
* Actor model – good for scalability

## Inventing a good async runtime

How to make parallelism for I/O bound applications better?

1. Wrap our business code into blocks: one block type for computations, another block type for non-blocking I/O, and one more for blocking. Write program as a connected chain of such blocks, in the way that allow us to put these blocks on a system thread for execution, and take them off on I/O operations. In facts it is a “green threads” (fibers) runtime.
2. Prefer non-blocking I/O API over blocking.
3. Create a separate thread/single thread pool with the highest priority for handling newly arrived data in selector/epoll. After the data is fetched, we can put the fiber that made the I/O call back onto the system thread to continue the execution.
4. For I/O calls that don’t have a non-blocking version (e.g. DNS calls), create an unbound cached thread pool.
5. For computation blocks use a fixed size thread pool that has same size as the number of CPU cores.

We can roughly say we have invented a Tokio runtime. And actually Cats-Effects runtime.
