---
title: Fork/Join框架使用与分析
toc: true
date: 2016-12-05 10:11:27
tags:
- Java
categories:
- Java并发编程
---

Fork/Join框架，用来解决某些特定任务，而这些任务可以分解成小任务，去执行小任务，合并结果。

## 样例一
价格更新，mock批量的价格类（Product），然后创建任务Task（继承RecursiveAction），将Task由ForkJoinPool执行，更新价格。

### 样例代码
Product类
~~~java
public class Product {

	private String name;
	private double price;
    ...
}
~~~

ProductListGenerator类，mock指定大小的product list，
~~~java
public class ProductListGenerator {

	public List<Product> generate(int size) {
		List<Product> ret = new ArrayList<Product>();
		for (int i = 0; i < size; i++) {
			Product product = new Product();
			product.setName("Product " + i);
			product.setPrice(10);
			ret.add(product);
		}
		return ret;

	}

}
~~~

Task类，继承RecursiveAction类，该任务没有返回结果，
~~~java
public class Task extends RecursiveAction {

	private static final long serialVersionUID = 1L;

	private List<Product> products;

	private int first;
	private int last;
	private double increment;


	public Task (List<Product> products, int first, int last, double increment) {
	    this.products=products;
	    this.first=first;
	    this.last=last;
	    this.increment=increment;
	  }

	//Task被调度的方法，这里有拆分任务的逻辑
	@Override
	protected void compute() {
    	//直接计算
		if (last - first < 10) {
			updatePrices();
        //拆解任务
		} else {
			int middle = (last + first) / 2;
			System.out.printf("Task: Pending tasks: %s\n", getQueuedTaskCount());
			Task t1 = new Task(products, first, middle + 1, increment);
			Task t2 = new Task(products, middle + 1, last, increment);
			invokeAll(t1, t2);
		}

	}

	private void updatePrices() {
	    for (int i=first; i<last; i++){
	      Product product=products.get(i);
	      product.setPrice(product.getPrice()*(1+increment));
	    }
	  }

}
~~~

Main方法，
~~~java
public static void main(String[] args) {
    ProductListGenerator generator = new ProductListGenerator();
    List<Product> products = generator.generate(10000);
    Task task=new Task(products, 0, products.size(), 0.20);
    ForkJoinPool pool=new ForkJoinPool();
    pool.execute(task);

    do {
        System.out.printf("Main: Thread Count: %d\n", pool.getActiveThreadCount());
        System.out.printf("Main: Thread Steal: %d\n", pool.getStealCount());
        System.out.printf("Main: Parallelism: %d\n", pool.getParallelism());
        try {
            TimeUnit.MILLISECONDS.sleep(5);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    } while (!task.isDone());

    pool.shutdown();

    if (task.isCompletedNormally()) {
        System.out.printf("Main: The process has completed normally.\n");
    }

    for (int i = 0; i < products.size(); i++) {
        Product product = products.get(i);
        if (product.getPrice() != 12) {
            System.out.printf("Product %s: %f\n", product.getName(), product.getPrice());
        }
    }

    System.out.println("Main: End of the program.\n");


}
~~~

### 源码分析

ForkJoinPool
#### 默认构造
并行度：最大为32767（MAX_CAP），最小是运行CPU个数。
使用默认的线程工厂
异常处理，默认为null，没有处理
异步模式， true为FIFO，false为LIFO。背后的含义要分析。
ctl字段，在成功添加任务后，需要这个字段来做判断。如果并行度为4，ctl则为一个负数（-844442110001152）。

~~~java
public ForkJoinPool() {
    this(Math.min(MAX_CAP, Runtime.getRuntime().availableProcessors()),
         defaultForkJoinWorkerThreadFactory, null, false);
}

private ForkJoinPool(int parallelism,
                         ForkJoinWorkerThreadFactory factory,
                         UncaughtExceptionHandler handler,
                         int mode,
                         String workerNamePrefix) {
    this.workerNamePrefix = workerNamePrefix;
    this.factory = factory;
    this.ueh = handler;
    this.config = (parallelism & SMASK) | mode;
    long np = (long)(-parallelism); // offset ctl counts
    //这里，将ctl初始化为一个负数，这个负数转换成int后，是0。这数学的设计。。。
    this.ctl = ((np << AC_SHIFT) & AC_MASK) | ((np << TC_SHIFT) & TC_MASK);
}
~~~

#### 任务执行
ForkJoinPool提供execute方法，添加任务并执行。此为ForkJoinPool也是Executor，提供submit方法，支持ForkJoinTask和Runnable（Runnable包装成ForkJoinTask）两种task。
创建好ForkJoinPool后，就可以提交任务执行了。那么提交任务的逻辑中包括的初始化。

~~~java
public void execute(ForkJoinTask<?> task) {
    if (task == null)
        throw new NullPointerException();
    externalPush(task);
}
~~~

调用逻辑，ForkJoinPool.execute(task) --> externalPush(task) --> externalSubmit(task)。业务代码都集中在externalSubmit方法中。runState如果小于0，意味着ForkJoinPool已被终止，不接受任务了，抛出异常。
~~~java
if ((rs = runState) < 0) {
    tryTerminate(false, false);
    throw new RejectedExecutionException();
}
~~~

如果runState是0，说明未初始化，进行初始化，创建workQueues。workQueues的大小是CPU个数的两倍（自己根据代码算是这样的，不能保证正确）。
**这里只进行workQueues数组的初始化，提交的task还没处理。**
~~~java
else if ((rs & STARTED) == 0 ||     // initialize
         ((ws = workQueues) == null || (m = ws.length - 1) < 0)) {
    int ns = 0;
    //先锁上RSLOCK，通过Unsafe的CAS，锁上标记位RSLOCK
    rs = lockRunState();
    try {
    	//判断是否已经初始化，正常（不考虑并发）情况是0。
        if ((rs & STARTED) == 0) {
            U.compareAndSwapObject(this, STEALCOUNTER, null,
                                   new AtomicLong());
            // create workQueues array with size a power of two
            //默认情况，P是当前主机CPU单元的个数
            int p = config & SMASK; // ensure at least 2 slots
            int n = (p > 1) ? p - 1 : 1;
            n |= n >>> 1; n |= n >>> 2;  n |= n >>> 4;
            n |= n >>> 8; n |= n >>> 16; n = (n + 1) << 1;
            //自己算了下，如果CPU是4个，那么队列的size是8。
            workQueues = new WorkQueue[n];
            ns = STARTED;
        }
    } finally {
    	//unlock方法会更新runState, runState为4
        unlockRunState(rs, (rs & ~RSLOCK) | ns);
    }
}
~~~

externalSubmit方法中，弄了个死循环，所以逻辑还得继续走。再进行runState的状态判断逻辑。
如果runState还是为0，初始化一个WorkQueue。
~~~java
else if (((rs = runState) & RSLOCK) == 0) {
	// create new queue
    q = new WorkQueue(this, null);
    q.hint = r;
    q.config = k | SHARED_QUEUE;
    q.scanState = INACTIVE;
    rs = lockRunState();           // publish index
    if (rs > 0 &&  (ws = workQueues) != null &&
        k < ws.length && ws[k] == null)
        ws[k] = q;                 // else terminated
    unlockRunState(rs, rs & ~RSLOCK);
}
~~~

最后一种情况，当workQueus数组的指定下标()不为null，将task放入这个WorkQueue。
~~~java
//为嘛是k = r & m & SQMASK，再研究
else if ((q = ws[k = r & m & SQMASK]) != null) {
	//给当前的队列加锁
    if (q.qlock == 0 && U.compareAndSwapInt(q, QLOCK, 0, 1)) {
        ForkJoinTask<?>[] a = q.array;
        int s = q.top;
        boolean submitted = false; // initial submission or resizing
        try {                      // locked version of push
            if ((a != null && a.length > s + 1 - q.base) ||
            	//growArray方法进行initial或者双倍扩容
                (a = q.growArray()) != null) {
                int j = (((a.length - 1) & s) << ASHIFT) + ABASE;
                U.putOrderedObject(a, j, task);
                U.putOrderedInt(q, QTOP, s + 1);
                submitted = true;
            }
        } finally {
            U.compareAndSwapInt(q, QLOCK, 1, 0);
        }
        //如果task成功提交，会尝试启动worker，来看看是怎么设计的
        if (submitted) {
            signalWork(ws, q);
            return;
        }
    }
    move = true;                   // move on failure
}
~~~

signalWork方法，添加task成功后调用。tryAddWorker方法会创建一个ForkJoinWorkerThread，并绑定这个ForkJoinPool和其中的一个队列WorkQueue。
~~~java
final void signalWork(WorkQueue[] ws, WorkQueue q) {
    long c; int sp, i; WorkQueue v; Thread p;
    //这里初始化后的ctl是个负数，但是转换成int后，变成0，很巧的数学设计
    while ((c = ctl) < 0L) {                       // too few active
        if ((sp = (int)c) == 0) {                  // no idle workers
            if ((c & ADD_WORKER) != 0L)            // too few workers
                //初始状态，会调用这个方法，调用完退出
                tryAddWorker(c);
            break;
        }
        if (ws == null)                            // unstarted/terminated
            break;
        if (ws.length <= (i = sp & SMASK))         // terminated
            break;
        if ((v = ws[i]) == null)                   // terminating
            break;
        int vs = (sp + SS_SEQ) & ~INACTIVE;        // next scanState
        int d = sp - v.scanState;                  // screen CAS
        long nc = (UC_MASK & (c + AC_UNIT)) | (SP_MASK & v.stackPred);
        if (d == 0 && U.compareAndSwapLong(this, CTL, c, nc)) {
            v.scanState = vs;                      // activate v
            if ((p = v.parker) != null)
                U.unpark(p);
            break;
        }
        if (q != null && q.base == q.top)          // no more work
            break;
    }
}
~~~

## 总结
回到最初的使用ForkJoinPool的代码，new一个，然后execute添加任务。我们分析了ForkJoinPool初始化过程。但是具体的执行，要在继续分析，看源码。

### 遗留问题
理解ThreadLocalRandom.getProbe()和ThreadLocalRandom.advanceProbe(r)的使用，这个方法并非公开的API。