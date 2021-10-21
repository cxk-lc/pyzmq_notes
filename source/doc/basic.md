# 基础篇

## REQ-REP模式

从一段简单传统的`Hello World!`程序开始。需要创建一个客户端和一个服务端，客户端发送`Hello!`，服务端发送`World!`。

### 服务端代码

在`5555`端口开启一个`ZMQ`的`socket`，等待请求，收到后回复

```python
# -*- coding: utf-8 -*-
import time
import zmq


def zmq_server(rep_msg: str = 'World!', ip: str = '127.0.0.1',
               port: int = 5555):
    context = zmq.Context()
    socket = context.socket(zmq.REP)
    socket.bind(f"tcp://{ip}:{port}")

    print('Waiting for user access...')
    #  Wait for request from client
    message = socket.recv()
    print(f"Received request: {message.decode()}")

    #  Do some 'work'
    time.sleep(1)

    #  Send reply back to client
    socket.send(rep_msg.encode())


if __name__ == '__main__':
    zmq_server()
```

![REQ-REP](..\img\Basic_imgs\image-20211020112808034.png)

使用`REQ-REP`模式的`socket`发送和接受消息是需要遵循一定规律的。客户端首先使用`send()`发送消息，再用`recv()`接收，如此循环。如果打乱了这个顺序（如连续发送两次）则会报错。类似地，服务端必须先进行接收，后进行发送。

### 客户端代码

```python
# -*- coding: utf-8 -*-
import zmq


def zmq_client(req_msg: str = 'Hello!', ip: str = '127.0.0.1',
               port: int = 5555):
    context = zmq.Context()

    #  Socket to talk to server
    print("Connecting to hello world server…")
    socket = context.socket(zmq.REQ)
    socket.connect(f"tcp://{ip}:{port}")

    print("Sending request …")
    socket.send(req_msg.encode())

    #  Get the reply.
    message = socket.recv()
    print(f"Received reply {req_msg} [ {message.decode()} ]")


if __name__ == '__main__':
    zmq_client()
```

理论上，一个服务器可以同时被很多个客户端连接。并且先打开客户端，再打开服务端也没有问题。

无论是客户端还是服务端，流程基本上一致，只是收发消息这个动作的差别：

1. 创建 `ZMQ`上下文
2. 服务端创建`REP`的`socket` / 客户端创建`REQ`的`socket`
3. 服务端将`REP`的`socket`绑定到指定端口上 / 客户端将 `REQ`的`socket`连接到服务端的端口上
4. 服务端等待客户端发送请求并响应 / 客户端给服务端发送请求并等待回复

## PUB-SUB模式

第二种经典的消息模式是单向数据分发：服务端将更新事件发送给一组客户端。比如天气`APP`将天气信息分发给用户。

### 发布者代码

```python
# -*- coding=utf-8 -*-

import zmq
import time


def zmq_pub(msg: str = 'World!', ip: str = '*', port: int = 5555):

    context = zmq.Context()
    socket = context.socket(zmq.PUB)
    socket.bind(f"tcp://{ip}:{port}")

    for i in range(15):
        if i < 10:
            i = f'0{i}'
        print(f'{i} send message: {msg}' + str(i))
        socket.send(f'{i} message {msg}'.encode())
        time.sleep(0.5)


if __name__ == '__main__':
    zmq_pub()

```

这里的消息发布，只要没有发布完，就不会停。

![PUB-SUB](..\img\Basic_imgs\image-20211020134322398.png)

### 订阅者代码

```python
# -*- coding: utf-8 -*-

import zmq


def zmq_sub(msg='Hello!', ip: str = '*', port: int = 5555):

    context = zmq.Context()
    socket = context.socket(zmq.SUB)
    socket.connect(f"tcp://{ip}:{port}")
    socket.setsockopt(zmq.SUBSCRIBE, b'00')  # Subscribe to messages starting with the '00' character
    # socket.setsockopt(zmq.SUBSCRIBE, b'')

    while True:
        print('Waiting for news release...')
        response = socket.recv()
        print("Response: %s" % response)


if __name__ == '__main__':
    zmq_sub()
```

使用`PUB-SUB`模式需要注意的一些问题：

- `PUB-SUB`模式的客户端，必须用`socket.setsockopt`来设置订阅的内容，否则什么都接收不到。如果想订阅全部的消息，可以将订阅设置成空字符串。订阅信息可以是任何字符串，可以设置多 次。只要消息满足其中一条订阅信息，SUB套接字就会收到。
- 在`PUB-SUB`模式中的`socket`是异步的。发布者可以在一个循环体里一直用`socket.send`去发布消息，但是如果尝试用`socket.recv`去接收信息就会报错；类似的，订阅者可以在一个循环体里一直使用`socket.recv`去接收信息，如果尝试用`socket.send`去发送消息，一样也是报错。
- 就算是先打开了`SUB`的`socket`，后打开`PUB`的`socket`去发送消息，这时`SUB`还是会丢失一些消息的，因为建立连接是需要一些时间的。很少， 但并不是零。

上述注意事项的第三点中描述的问题，一般称之为`slow joiner`。这种看似“慢连接”的症状其实很好理解。`ZMQ`这里是在后台进行异步的`I/Q`传输的，如果你的两个节点按照如下顺序连接：

1. 订阅者连接至端点接收消息并计数
2. 发布者绑定至端点并立刻发送`1000`条消息

在一些相对比较极端的情况下，订阅者可能一条消息都收不到，遇到这种情况，很多人下意识的是去检查是不是没有设置订阅消息，然后再重试。

我们知道，`TCP`连接在建立时需要进行三次握手，会消耗个几毫秒的时间，当然在这个过程中，节点的数量越多，需要的时间也会增加。即使是在这几毫秒的时间里，`ZMQ`也是可以发送出去很多信息了。假设建立连接一共花了`5`毫秒，而`ZMQ`可能只需要`1`毫秒就可以把这`1000`条消息全部发送出去。

只有当订阅者完全准备好时，发布者才发送消息，这样才能使两边完全同步，确保不会出现因为建立连接的耗时而丢消息。这里先说一种简单的方式来同步`PUB`和`SUB`，就是让`PUB`在发送消息之前`sleep`一下再发送。这种简单的方法，可以在测试`demo`时使用，实际的代码中因为不清楚网络状况，无法精准的控制`sleep`的时间。后面再谈如何真正解决。

`PUB-SUB`模式有如下特点：

- 订阅者可以连接多个发布者，轮流接收消息
- 如果发布者没有订阅者与之相连，那它发送的消息将直接被丢弃
- 如果你使用`TCP`协议，那当订阅者处理速度过慢时，消息会在发布 者处堆积（这里可以考虑设置阈值来保护发布者）
- 在目前版本的`ZMQ`中，消息的过滤是在订阅者处进行的。也就是 说，发布者会向订阅者发送所有的消息，订阅者会将未订阅的消息丢弃

## PUSH-PULL模式

又叫做`pipeline`管道模式，把数据交给一组worker端干活，`PUSH`会把任务均匀的（这个好像是`zmq`的招牌）的分配给下游的`worker`们，保证大家都有活干。

模型描述：

1. 上游发布任务
2. 中游执行任务
3. 下游结果收集

### 上游任务发布代码

```python
# -*- coding: utf-8 -*-

import random
import time
import zmq


def zmq_push_ventilator(ip: str = '127.0.0.1', port: int = 5555):
    context = zmq.Context()

    socket = context.socket(zmq.PUSH)
    socket.bind(f'tcp://{ip}:{port}')

    print('Are workers ready(yes/no): ', end='')
    while input().lower() != 'yes':
        time.sleep(0.1)
        print('Are workers ready(yes/no): ', end='')

    print('Sending tasks to workers...')
    
    socket.send('0'.encode()) # The first message is "0" and signals start of batch

    # Initialize random number generator
    random.seed()

    total_msec = 0
    for _ in range(100):
        # Random workload from 1 to 100 msecs
        workload = random.randint(1, 100)
        total_msec += workload
        socket.send(str(workload).encode())
    print(f'Total expected cost: {total_msec}')


if __name__ == '__main__':
    zmq_push_ventilator()
```

### 中游任务执行代码

```python
# -*- coding: utf-8 -*-

import sys
import time
import zmq


def zmp_pull_push_worker(ip: str = '127.0.0.1', port_up: int = 5555,
                         port_down: int = 5556):
    context = zmq.Context()

    # Socket to receive messages on
    ventilator_socket = context.socket(zmq.PULL)
    ventilator_socket.connect(f"tcp://{ip}:{port_up}")

    # Socket to send messages to
    sink_socket = context.socket(zmq.PUSH)
    sink_socket.connect(f"tcp://{ip}:{port_down}")

    while True:
        up = ventilator_socket.recv().decode()

        sys.stdout.write('#')
        sys.stdout.flush()

        time.sleep(int(up) * 0.001)  # Pretend to be working

        # Send results to sink
        sink_socket.send(f'work {up} is ok!'.encode())


if __name__ == '__main__':
    zmp_pull_push_worker()
```

### 下游结果接收代码

```python
# -*- coding: utf-8 -*-
import sys
import time
import tqdm

import zmq


def zmq_pull_sink(ip: str = '127.0.0.1', port: int = 5556):
    context = zmq.Context()

    # Socket to receive messages on
    socket = context.socket(zmq.PULL)
    socket.bind(f"tcp://{ip}:{port}")

    # Wait for start of batch
    _ = socket.recv()

    # Start our clock now
    t_start = time.time()

    # Process 100 confirmations
    total_msec = 0
    for _ in tqdm.tqdm(range(100), desc='Processing'):
        socket.recv()
    t_end = time.time()

    print(f'Total elapsed time: {(t_end - t_start) * 1000}')


if __name__ == '__main__':
    zmq_pull_sink()
```

该模式的使用要注意一下几个细节：

- `worker`上游与任务发布相连，下游与结果收集相连，这就意味着可以开启任意多个`worker`，这样也意味着，需要准备多个`worker`绑定用的断点。
- 在启动整个流程开始下发任务之前，一定要等待所有的`worker`已经启动。连接`socker`会花费一些时间，且一旦第一个`worker`连接成功，它就会一下子收到大量的任务。也就是说，所有的`worker`不能在任务下发前，全部完成启动，这些任务就不会被并行的执行。
- 任务发布的时候，会向所有已连接的`worker`均匀的发送任务，也就是负载均衡。

管道模式有时也会出现任务会让人误以为这个

管道模式与`PUB-SUB`模式一样都是单向的，但是有两点区别：

- 该模式下在没有消费者的情况下，发布者的信息是不会消耗的（由发布者进程维护）
- 多个消费者消费的是同一列信息，假设`A`得到了一条信息，则`B`将不再得到（任务发布不会重复）

这种模式主要针对在消费者能力不够的情况下，提供的多消费者并行消费解决方案。

![PUSH-PULL](..\img\Basic_imgs\image-20211020163359728.png)

## 正确的使用上下文

`ZMQ`应用程序的一开始总是会先创建一个上下文，并用它来创建`socket`。在`Python`中，创建上下文的函数是`zmq.Context()`。一个进程里只应该有一个上下文。上下文应该是一个容器，包含了该进程中所有的`socket`。如果一个进程中创建了两个上下文，那就相当于启动了两个`ZMQ`实例。如果这正是你的场景所必需的，那没有问题，但是一般情况下，一个进程里只应该有一个上下文。
