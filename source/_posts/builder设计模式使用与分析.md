---
title: builder设计模式使用与分析
toc: true
date: 2017-01-06 15:52:19
tags:
- Java
categories:
- 设计模式
---

最近工作中，一直使用到builder模式，记录一下。

## 定义
这个builder设计模式好像不属于四人帮提的23种，但不妨碍它的套路。builder，我的理解是构建。构建什么？构建Model。什么是Model?可以是简单的POJO，也可以是包含复杂业务的Business Model。我这里涉及的是POJO。

使用builder模式，保证Model的不可变性，避免其他地方恶意修改。


## 使用场景

### Web请求参数多变
这里我自己的经验，是在web服务中，一个请求拥有太多参数（比如超过5个），而且随着业务的精进，可能还会扩展更多的参数，那么可以考虑builder模式，封装变化。

### Web Service服务请求体包装
在系统中，需要经常调用数据服务，传统的SOAP服务，亦或RestFull服务。通常需要提供request body。如果request的处于变化之中，想要便于扩展，可以考虑builder模式。也可以跳出这个思路，Web Service做版本控制。


## 样例

这里有个需求，提供一个Http Get服务，根据提供的参数生成PDF文档。这里参数可能随便支持的业务而扩展。对于生成PDF的http服务背后，对应的是Service接口，那么直接将Http请求的参数，作为Service接口的形参，是不友好的。可以创建一个PDFRequest，封装请求参数的变化，扩展时，只要更新PDFRequest的builder类。对于Service接，适配PDFRequest，这样就屏蔽的变化。

- Model构造函数私有
- 只提供Get方法
- 静态内部Builder类
- 内部类build方法创建Model


~~~java
public class PDFRequest {

	private final String xxx;

	//构造函数私有
	private PDFRequest(PDFRequestBuilder builder){
		this.xxx = builder.xxx;

	}

    //builder类
	public static class PDFRequestBuilder {
		private final long xxx;

		public PDFRequestBuilder(){
		}

		public PDFRequestBuilder xxx(String xxx){
			this.xxx = xxx;
			return this;
		}

	}

    //只提供Get方法
	public long getxxx() {
		return xxx;
	}


}
~~~
