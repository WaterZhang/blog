---
title: Fork/Join框架使用与分析二
toc: true
date: 2016-12-13 15:50:20
tags:
- Java
categories:
- Java并发编程
---

上一篇介绍了Fork/Join的无返回任务，并尝试解读源码，发现用了很多巧妙的位运算，理解源码时，比较费劲，放弃源码分析。

## 样例二
模拟在文档中查找关键字

### Document类
模拟文档内容
~~~java
public class Document {
	//模拟的词组
	private String words[] = {"the","hello","goodbye",
			"packt","java","thread","pool","random","class","main"};
	//创建二维数组
	public String[][] generateDocument(int numLines, int numWords, String word) {
		int counter = 0;
		String document[][] = new String[numLines][numWords];
		Random random = new Random();
		for (int i = 0; i < numLines; i++) {
			for (int j = 0; j < numWords; j++) {
				int index = random.nextInt(words.length);
				document[i][j] = words[index];
                //记录查找关键字的个数，方便运行结束后进行比对
				if (document[i][j].equals(word)) {
					counter++;
				}
			}
		}
		System.out.println("DocumentMock: The word appears "+ counter+" times in the document");
	    return document;
	}
}

~~~

### DocumentTask
继续RecursiveTask，支持返回类型。这个任务解决一维（文档）的问题，二维（行）的问题交给LineTask来执行。任务类要有分解任务的逻辑。

~~~java
public class DocumentTask extends RecursiveTask<Integer> {

	private static final long serialVersionUID = 1L;
	
	private String document[][];
	private int start, end;
	private String word;

	public DocumentTask(String document[][], int start, int end, String word) {
		this.document = document;
		this.start = start;
		this.end = end;
		this.word = word;
	}

	@Override
	protected Integer compute() {
		int result = 0;
        //符合特定条件，不用分解任务，这里将计算交给的另外的任务
		if (end - start < 10) {
			result = processLines(document, start, end, word);
		} 
        //分解任务，合并计算结果
        else {
			int mid = (start + end) / 2;
			DocumentTask task1 = new DocumentTask(document, start, mid, word);
			DocumentTask task2 = new DocumentTask(document, mid, end, word);
			invokeAll(task1, task2);
			try {
				result = groupResults(task1.get(), task2.get());
			} catch (InterruptedException | ExecutionException e) {
				e.printStackTrace();
			}
		}
		return result;
	}

	private Integer processLines(String[][] document, int start, int end, String word) {
		List<LineTask> tasks = new ArrayList<LineTask>();
		for (int i = start; i < end; i++) {
			LineTask task = new LineTask(document[i], 0, document[i].length, word);
			tasks.add(task);
		}
		invokeAll(tasks);
		int result = 0;
		for (int i = 0; i < tasks.size(); i++) {
			LineTask task = tasks.get(i);
			try {
				result = result + task.get();
			} catch (InterruptedException | ExecutionException e) {
				e.printStackTrace();
			}
		}
		return new Integer(result);
	}

	private Integer groupResults(Integer number1, Integer number2) {
		Integer result;
		result = number1 + number2;
		return result;
	}

}
~~~

### LineTask
用来解决行中查找关键字的任务

~~~java
public class LineTask extends RecursiveTask<Integer> {

	private static final long serialVersionUID = 1L;
	private String line[];
	private int start, end;
	private String word;

	public LineTask(String line[], int start, int end, String word) {
		this.line = line;
		this.start = start;
		this.end = end;
		this.word = word;
	}
	
	@Override
	protected Integer compute() {
		Integer result = null;
		if (end - start < 100) {
			result = count(line, start, end, word);
		} else {
			int mid = (start + end) / 2;
			LineTask task1 = new LineTask(line, start, mid, word);
			LineTask task2 = new LineTask(line, mid, end, word);
			invokeAll(task1, task2);
			try {
				result = groupResults(task1.get(), task2.get());
			} catch (InterruptedException | ExecutionException e) {
				e.printStackTrace();
			}
		}
		return result;
	}
	
	private Integer count(String[] line, int start, int end, String word) {
		int counter;
		counter = 0;
		for (int i = start; i < end; i++) {
			if (line[i].equals(word)) {
				counter++;
			}
		}
		try {
			Thread.sleep(10);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		return counter;
	}
	
	private Integer groupResults(Integer number1, Integer number2) {
		Integer result;
		result = number1 + number2;
		return result;
	}
}
~~~

### 测试类
~~~java
public static void main(String[] args) {
    Document mock = new Document();
    String[][] document = mock.generateDocument(100, 1000, "the");
    DocumentTask task = new DocumentTask(document, 0, 100, "the");
    ForkJoinPool pool = new ForkJoinPool();
    pool.execute(task);
    //等待任务完成
    do {
        System.out.printf("******************************************\n");
        System.out.printf("Main: Parallelism: %d\n", pool.getParallelism());
        System.out.printf("Main: Active Threads: %d\n", pool.getActiveThreadCount());
        System.out.printf("Main: Task Count: %d\n", pool.getQueuedTaskCount());
        System.out.printf("Main: Steal Count: %d\n", pool.getStealCount());
        System.out.printf("******************************************\n");
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    } while (!task.isDone());

    pool.shutdown();
    //等待任务关闭完成
    try {
        pool.awaitTermination(1, TimeUnit.DAYS);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    try {
        System.out.printf("Main: The word appears %d in the document", task.get());
    } catch (InterruptedException | ExecutionException e) {
        e.printStackTrace();
    }
}
~~~

## 总结

这是第二篇介绍Fork/Join，但是没有花时间再去研究源码，付出和收获不成正比，大概知道Fork/Join的基本原理，理解背后的设计。这里有篇[参考文章](http://www.infoq.com/cn/articles/fork-join-introduction)。