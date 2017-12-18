# fuzzing-test
# Fuzzing Test及在Display的实践

Fuzzing Test，又叫模糊测试，想必很多人都听过，但又不知道如何来做，这篇文章是我们在后台模块上运用fuzzing技术的一个总结，可以帮助没接触过又有兴趣的同学快速上手。

## Fuzzing Test概述

### Monkey Test
你一定听说过这样一个理论：
> 让一只猴子在打字机上随机地按键，当按键时间达到无穷时，几乎必然能够打出任何给定的文字，比如莎士比亚的全套著作。

这个理论叫做无限猴子定理。测试中常见的 [Monkey Test](https://en.wikipedia.org/wiki/Monkey_testing) 中 Monkey 一词，正是来源于此。Monkey Testing，是指通过随机的向程序输入数据，并监控程序的行为，以发现程序中的bug，例如 crash、内存泄露等。有人可能会说，Monkey Test不就是在穷举吗，这种简单的方法也带了很大的局限性，纯随机生成的数据中，大部分测试数据覆盖的是重复的代码路径，这导致效率十分低下。

### Fuzzing Test
Monkey Test 思路是对的，但有没有更高效的穷举方法呢？当然是有的，现在开始介绍我们的主角：[Fuzzing Test](https://en.wikipedia.org/wiki/Fuzzing)。Fuzzing Test的核心思想是:
现实世界中，程序的输入数据大部分是存在某种特定语义的，基于一组有效语义的 seed 数据以及各种 Fuzzing Strategy 所生成的测试数据，更容易覆盖更多有效的代码路径。例如我们在为一个图片浏览器程序生成数据时，如果是纯粹随机的数据，那么大部分的测试数据都是无效的，会因文件不符合某种格式而走入相同的代码路径。这显然极其低效。而基于有效的图片文件进行比特翻转，或文件大小调整，则更容易避免这种情况，从而尽可能高效的生成有效测试数据。

### 为什么要做Fuzzing Test
Microsoft 在[文章](https://patricegodefroid.github.io/public_psfiles/talk-issta2010.pdf) 提到，从2008年开始，他们部署了 100+ 台服务器，7/24 小时运行 Fuzz Testing，用于对各种 parser 类程序进行测试，包括图片处理器，媒体播放器等等。Google 开源了他们的 Fuzz Testing 系统 [OSS Fuzz](https://github.com/google/oss-fuzz/blob/master/docs/clusterfuzz.md) ，据称也有数百台虚拟机，用于对 Chrome 浏览器进行测试。为什么Microsof和Google会投入这么大的精力来做Fuzzing呢，因为Fuzzing能帮助发现关于安全的bug，一旦安全漏洞被攻击，可能导致最严重的经济损失。

## Peach Fuzz工具
当前fuzzing Test的工具很多，有ALF、EasyFuzzer、DFUZ、SPIKE、Google bunny等，我们从中选用了一种支持Smart Fuzz的Fuzzing工具Peach，主要基于以下三个原因：

1. 有成熟的开源版本，扩展性好。
2. 强大，Peach支持常见的fuzz对象，如网络协议、文件格式等。只需要根据自己的需求定义data model即可。
3. Peach提供端到端的功能，Fuzzing包含了监控、测试前校验、代码覆盖率等功能。


### Peach的一些重要概念
* __Modeling__</br>
    在Peach里，用户要对数据（Data Model）和状态 (State Model) 建模。首先，要抽象出fuzzing对象的数据描述。比如说，我要对一个http server做fuzzing test，那么要fuzzing的数据就http协议，在Data Model中用户就可以定义适用于自己场景的http协议数据。有了fuzzing数据后，State Model要干的事就是定义如何发送和接收fuzzing数据到fuzzing目标。
* __Publisher__</br>
    Publiser是I/O接口，他们把State Model中看到的输入，输出，调用等抽象概念提供给实际的传输或实现。Peach包含许多发布者，这些发布者可以写入文件，通过TCP、UDP或其他网络协议进行连接，进行Web请求，甚至可以调用COM对象。
* __Fuzzing Strategy__</br>
    用户可以自定义fuzzing的策略，比如一次是变异一个元素还是多个元素，某部分变异次数是否比其他部分要更多，使用哪个mutator等。
* __Mutators__</br>
    Fuzzing Strategy定义了变异的基本规则，mutators生成最终的数据。mutators使用现有的默认值并对其进行修改，或者生成全新的数据。比如，生成的数据是：“从当前值生成数字 - 50到当前值+ 50”。或“生成长度在1到10,000个字符之间的字符串”。
* __Agents__</br>
    Agents是可以在本地或远程运行的特殊Peach进程，可以承载一个或多个monitors。在Peach中，要单独通过python peach.py -a起一个agent进程，默认在9000端口监听。
* __Monitors__</br>
    监视器运行在agent进程的内部，是真正任务的执行者。例如，在fuzzing过程中捕获网络流量，或者将调试器附加到进程以检测崩溃，甚至在崩溃或停止时重新启动网络服务。

## Display实践
Display是广点通播放系统的最外层，负责将广点通的对外接入协议转换为合法的内部处理协议，发起广告请求，并将待播广告处理后，返回给流量方。Display能够处理各种非法请求的能力对整个广点通都至关重要，因此，我们首选对display进行fuzzing test。

Display是一个http server，能处理来自微信、浏览器、移动联盟以及RTB的http请求。我们要fuzzing各种异常的http请求，先看http协议的定义：
