# Reactor模式介绍  

Wikipedia上说：“The reactor design pattern is an event handling pattern for handling service requests delivered concurrently by one or more inputs. The service handler then demultiplexes the incoming requests and dispatches them synchronously to associated request handlers.”。从这个描述中，我们知道Reactor模式首先是事件驱动的，有一个或多个并输入源，有一个Service Handler，有多个Request Handlers；这个Service Handler会同步的将输入的请求（Event）多路复用的分发给相应的Request Handler。  

# Reactor模式的实现（以Redis为例）  

```
#事件主循环
def main():
	#初始化服务器，执行初始化任务像安装连接应答处理器（aeAcceptHandler）等
	init_server()
	#一直处理事件，直到服务器关闭为止
	while server_is_not_shutdown():
		aeProcessEvents()

	#服务器关闭，执行清理操作
	clean_server()
```

```
#处理各类事件
def aeProcessEvents():
	#获取到达事件离当前时间最接近的时间事件
	time_event = aeSearchNearestTimer()
	#计算接近的时间事件距离到达还有多少毫秒
	remaind_ms = time_event.when - unix_ts_now()
	#如果时间事件已到达，那么remaind_ms的值可能为负数，将它设置为零
	if remaind_ms < 0:
		remaind_ms = 0
	#根据remaind_ms的值，创建timeval结构
	timeval = create_timeval_with_ms(remaind_ms)
	#阻塞并等待文件事件的产生，最大阻塞时间由传入的timeval结构决定
	#如果remaind_ms的值为0，那么aeApiPoll调用之后马上返回，不阻塞
	aeApiPoll(timeval)

#处理所有已产生的文件事件
processFileEvents()  

#处理所有已到达的时间事件
processTimeEvents()
```

```
#处理时间事件
def processTimeEvents():
	#遍历服务器中所有的时间事件
	for time_event in all_time_event():
		#检查事件是否已经到达
		if time_event.when <= unix_ts_now():
			#事件已到达，执行事件处理器，并获得返回值
			retval = time_event.timeProc()
			#如果是一个定时事件
			if retval == AE_NOMORE:
				#那么将该事件从服务器中删除
				delete_time_event_from_server(time_event)
			#如果是周期性事件
			else:
			#那么按照事件处理器的返回值更新事件事件的when属性
			#让这个事件在指定时间之后再次到达
			update_when(time_event, retval)

#处理文件事件
```
* 当套接字变得可读时（客户端对套接字执行write操作，或者执行close操作），或者有新的可应答（acceptable）套接字出现时（客户端对服务器的监听套接字执行connect操作），套接字产生AE_READABLE事件。  
* 当套接字变得可写时（客户端对套接字执行read操作，或者执行close操作），套接字产生AE_WRITABLE事件。  

```
def processFileEvents():
	#如果是AE_READABLE事件
	if（aeFileEvent & AE_READABLE):
		#执行读事件处理器
		aeFileEvent->rFileProc()
	#如果是AE_WRITABLE事件
	if (aeFileEvent & AE_WRITABLE):
		#执行写事件处理器
		aeFileEvent->wFileProc()
```

​	