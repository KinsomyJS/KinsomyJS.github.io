## 第一章

### Happen-Before规则
指令重排的原则：

* 程序顺序原则：一个线程内保证语义的串行性

* volatile规则：volatile变量的写先发生于读，保证volatile变量的可见性

* 锁规则：解锁unlock必然发生在随后的加锁lock之前

* 传递性：A先于B，B先于C，那么A必然先于C

* 线程的start()先于线程的每一个动作

* 线程的所有操作先于线程的join()发生

* 线程的中断（interrupt()）先于被中断线程的代码

* 对象的构造函数执行、结束先于finalize()方法

## 第二章
### 1.新建线程
```java
//正确
Thread t1 = new Thread();
t1.start();

//错误
Thread t2 = new Thread();
t2.run();
```
新建线程需要调用线程的start()方法，方法会新建一个线程并让新建线程执行run()方法，如果直接使用t2.run()方法，会编译通过，也能正常执行，但是它不会新建一个线程，而是在当前线程调用run(),串行执行。

### 2.终止线程

stop()被标记为废弃，因为该方法太过暴力，会强行终止正在执行的线程，并且释放线程所持有的锁，会引发数据的不一致。

### 3.中断线程
```java
// 中断线程
public void Thread.interrupt()               // 判断是否被中断
public boolean Thread.isInterrupted()        // 判断是否被中断，并清除当前中断状态
public static boolean Thread.interrupted()   
```

sleep()方法会抛出InterruptedException中断异常，当在sleep过程中被中断就会抛出，此时会清除中断标记，因此在捕获异常处理时，要手动再次设置中断标记。
```java
Thread t1 = new Thread(){
	@Override
	public void run() {
		while (true){
			if(Thread.currentThread().isInterrupted()){
				break;
			}
			try {
				Thread.sleep(1000);
			} catch (InterruptedException e) {
				//中断标记被清除，需要重新设置
				Thread.currentThread().interrupt();
			}
			Thread.yield();
		}
	}
};
t1.start();
t1.interrupt();
```


