# queuegroup
使用go实现一个分组排队取号叫号的逻辑

# 业务场景（这个包产生的原因）
1. 在我们的项目中，由同一个客服端发送到服务器的请求，必须按照请求的顺序进行处理（数据只存在服务端，下一个请求的基础数据就是上一个请求运算完成后的结果）；
2. 无法保证客服端请求一定严格收到返回后才发送下个请求，偶尔客户端会短时间内发送多个请求，如果直接并发处理会导致数据错误或丢失（多个请求使用旧数据进行运算）。

# 思路
1. 由于请求需要的基础数据依赖于上一次请求的结果，自然想到需要将请求串型处理，说到串型处理，那就是队列了，由于我们只需要对同一客户端发送的请求进行串型处理，所以就需要分组队列，每个组单独一个队列即可。
2. 在我们的需求中，主要是每个请求的基础数据依赖于上一个请求，那么只需要在当前请求运算结束以前阻塞后面的请求获取基础数据即可，也就是在当前请求返回前，独占这个队列，不让后面的请求使用。

基于上面两点，我联想到了车站排队购票或者医院排队挂号等线下排队的情形，基本和我们的需求相同，再结合银行的取号排队方式，实现一个分组队列方法：
* 1个请求 >>>>>> 一个需要办理业务的人
* 请求需要的基础数据 >>>>>> 柜台自助机
* 请求入队 >>>>>> 排队取号
* 独占基础数据 >>>>>> 霸占自助机
* 请求离队 >>>>>> 释放自助机

逻辑步骤如下：
1. 进入窗口（一个组）取号（窗口不存在则新开窗口）
2. 等待该窗口叫号
3. 被叫到号的人（一个请求）开始办理业务
4. 办理完成后（或超时）离开该窗口（组）
5. 继续叫号
6. 若该窗口一段时间（闲置时间）没有人取号办理业务，关闭该窗口以节约资源（也可以不关闭）

# 使用方法
基本使用方法：
```go
package main

import (
	"fmt"
	"sync"

	"github.com/zboyco/queuegroup"
)

func main() {
	wg := sync.WaitGroup{}
	// 配置超时和过期时间
	queuegroup.Config(
		100, // 单个队列最大长度（默认10）
		500, // 单个号业务办理超时时间（毫秒，默认不超时）
		30,  // 组队列没有排号后多长时间关闭队列（秒，默认不关闭）
	)
	// 获取队列
	queue := queuegroup.GetQueue(0) // 传入队列组ID

	for i := 0; i < 100; i++ {
		wg.Add(1)

		// 取号
		ticket := queue.QueueUp()

		go func(mt *queuegroup.Ticket, id int) {
			// 等待叫号
			mt.Wait()

			// 办理业务
			fmt.Printf("办理成功: %v \n", id)

			// 离开队伍，下一个才能被叫号（或者超时）
			err := mt.Leave()
			if err != nil {
				fmt.Println(err)
			}

			wg.Done()
		}(ticket, i)
	}
	wg.Wait()
}

```
* 上面的例子直接使用包级函数，当然也可以初始化一个单独的队列组实例，使用方法和包级函数相同：
```go
// 新建队列组
dataQueue := queuegroup.NewQueueGroup()
// 配置超时和过期时间
dataQueue.Config(10, 500, 30)
```