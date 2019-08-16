# 设计文档

- 题目：可插入式的PD调度器
- 成员：张航箔、唐玉迎、张思蒙

## 概要

设计了一个可以用户自定义的调度器，包含一些用户可以能有需求但PD之前没有的调度以及制定一些简单的抽象的规则让PD遵循该规则调度。

在PD中添加了新的用户可以使用的Scheduler，包括将Region移到指定的含有某个Label标签的Stores上，用户也可以通过Key或者表来指定Region移到指定的Label。用户也可以使用简单的抽象规则来让整个集群向用户定义的规则进行调度，例如将属于某个Label的Stores都减轻压力，或者将负载都移到属于某个Label的Stores上。

用户添加Scheduler之后会在系统中运行这个Scheduler，并保证系统定时的运行这个Scheduler，直到达到用户指定的调度状态。允许系统的调度在用户调度的范围内做调度，但是如果由用户调用某个region，那么五分钟之内该region不会被调度。

所有的用户命令传入都是基于PD-Ctl的客户端入口，所有命令都可以在PD运行时通过PD-Ctl客户端给API接口发送相应的请求，然后进行具体的调度。

## 背景

* 现有的PD Control已经包含很多功能，但对于用户来说，用户可能不知道自己所存数据的region ID、store ID，而仅仅关心整个Table或者Key范围的数据的放置，并将其使用Label来标记希望存储的Store。例如台风来了，用户希望将所有的表都从城市Label为上海的Stores移到城市Label为北京的Stores。因此我们在PD Control的基础上，允许用户添加新的使用Table，Key，Label标记的Scheduler，更好的满足用户需求。除了存储的位置外，目前PD也不允许用户进行时间相关的调度，例如用户直到晚上八点会迎来高峰使用，那么他希望在七点系统自动将那部分热点数据分散，因此我们加上了定时调度数据的策略。
* 现有的PD调度策略没有很好的解决一些冲突问题，当用户希望添加的Scheduler与系统Scheduler发生冲突时，就可能用户调过去然后又调回来的情形。不断的在调度，既没有达到用户希望的状态，系统系能也受到影响。所以我们让用户的调度在不严重影响系统性能的前提下尽可能的调度成功。
* 现有的PD并不能让用户指定所有的调度往某个规则的方向调度，例如：用户将位于北京的机房都换成了三星的硬盘，那么用户希望所有目前的位于东北的PD集群的Store都能够承担集群更多的负载，而其它地区的Stores能够减轻负载，那么就应该帮助PD的调度往这个方向调度。所以我们让PD原有的调度策略在不出现影响系统平衡的情况下遵守用户的规则。

## 方案

计划的PD调度器主要分为以下几个模块：

1. 客户端：作为用户传入参数的地方，用户通过客户端输入参数进行调度和制定规则。
2. 调度逻辑：用户传入具体调度的命令后，需要根据用户的调度策略生成具体的Operator，并通过gRpc发给对应的TiKV Store。
3. 冲突解决：如何保证用户的调度与系统的调度发生冲突之后，用户的调度能正常完成，用户的调度完成之后如何不影响系统的平衡而导致系统崩溃。
4. 规则设计：实现让用户指定一个抽象的规则，需要系统不断检测是否满足该规则，而不是一个个具体的Operator实现。

### 客户端

客户端作为用户命令的入口，我们的计划有两种：1.通过用户写入到一个配置文件，如果该配置文件被修改了就会发送一个信号给PD，PD不断侦听该信号，如果收到信号就去读取该配置文件。2.通过启动另外一个线程，收到命令行参数之后，直接通过PD的API接口控制PD进行调度。

最后我们选择了基于PD已有的PD-Ctl设计客户端，用户可以通过PD-Ctl查看命令选项并进行调度。

### 调度逻辑

#### 指定region和label

用户输入两个参数：region ID和希望调度的目的stores的label。如果该label的store只有一个或两个且该stores上没有该region的peers，则将其中一个或两个peers移到该stores上。如果该label的stores有三个或者三个以上，则在这些stores中挑选分数最低的作为目标store。

#### 指定table和label

用户知道自己将数据存放在哪一个` table`，希望将这个table中的数据移动到其他store或label下。用户输入两个参数：表名和希望调度的目的stores的label。找到该table上的所有region，然后对其中的每一个region，将他们的peers移到属于该label的store上。

####  指定key，范围和label

用户输入三个参数：key，该key之后需要调度多少个region，以及label。将该key所在的region之后的这些region都移到固定Label的stores上，逻辑如上。

## 基本原理

### Disadvantage

- 用户可能会有一些恶意调度
- 用户的调度可能会对系统调度产生影响

### Advantage

- 当用户调度和系统调度发生冲突的时候，不至于系统频繁的移动region
- 用户可以更加灵活的调度region

## 实现

### 参数传入

- 张思蒙
- 7.23-8.15
- step ：

### 调度实现

- 唐玉迎
- 7.23-8.15
- step ：

### 冲突解决

- 张航箔
- 7.23-8.15
- step ：

## 后续

现阶段我们先实现上面所述的四个方案，后续我们将根据进度和时间的安排，考虑怎样在策略执行后保持状态的问题以及控制器的相关问题。

# 功能测试

* 我们小组配置环境是：在一个物理机上起了一个PD、五个TIKV、一个TIDB
* 我们主要实现的是scheduler相关在scheduler下新加了t_add的子命令，在其下，分别是我们增加的功能：

![1](https://graph.baidu.com/resource/11179cbac01f3d1ab092a01565676786.jpg)


## 1. transfer-region-to-label
* 将指定的region挪到指定的label
### 用法：

![2](https://graph.baidu.com/resource/1110b08a59e4738293fe101565677208.jpg)

### 示例：
* 将region24，转移到label zone s1上
![3](https://graph.baidu.com/resource/111b4fab5d74928e0025501565677324.jpg)

* 添加scheduler成功
![4](https://graph.baidu.com/resource/111a1d11fe547154e681c01565677351.jpg)

* 若再次执行，程序会检查出该scheduler已存在，拒绝重复添加
![5](https://graph.baidu.com/resource/111afb1988a8bc7f08c5601565677404.jpg)

* 转移成功后的region24的一个peer已经转移到了label s1上
![6](https://graph.baidu.com/resource/111d449c5d7228c4cebdc01565677534.jpg)

* 当前已经没有任何scheduler，因此保证了确实是transfer-region-to-label将region24进行了迁移。
![7](https://graph.baidu.com/resource/111b6ddb0027299d413b701565677583.jpg)

* 上述操作后添加了一个scheduler
![8](https://graph.baidu.com/resource/111444b02b0326400104601565677695.jpg)


**transfer-region-to-label是我们所有scheduler调度的基础，后续每一个scheduler的添加到最后都会产生transfer-region-to-label操作，进而对label进行迁移**
## 2. transfer-region-to-store
* 迁移前的region17：
![9](https://graph.baidu.com/resource/1118599a13f5677f9612a01565677803.jpg)

* 添加scheduler，将region17的一个peer转移到store 5上
![10](https://graph.baidu.com/resource/1118387a57a29e6e228d701565677857.jpg)

* 迁移后的region17
![11](https://graph.baidu.com/resource/1117fe7eb5921ae7927b301565677898.jpg)


## 3. transfer-regions-of-label-to-label
### 作用：
* 将一个label下的的所有region转移到另一个label下。         
* 如果target label对应的store不止一个，则将source label下的region都转移过去；如果target label的store超过三个，则选择最优的三个进行迁移          
* 我们给出的示例中，每个label只对应一个store，所以转移时只转移region的一个peer        
* 若target label下已有source label下region的peer，则不迁移peer。
### 用法：
![12](https://graph.baidu.com/resource/11191b4a93d97a88b7fb701565677930.jpg)


### 示例：
* 看其中一个例子，region 43的副本之前在store6、store8、store5，它们的label分别s6、s8、s5
![13](https://graph.baidu.com/resource/111bd873feb12cfe815c801565678043.jpg)


* 将s6下的所有region转移到s1上，添加scheduler成功
![14](https://graph.baidu.com/resource/11108dea26a6714807c8d01565678086.jpg)





* 转移后region 43的副本就在store8、store5、store1上。
![15](https://graph.baidu.com/resource/111a42cdb0a75098a5a4f01565678136.jpg)

## 补充
* 在t_add下的scheduler中，凡是写了regions的都是可以将source的region挪动到几个target label下的。          
* 如果target label下有多个store，则可以挪一个region的多个副本到target label下。
### 示例：
* 当前label的K：zone，有两个V：s5、s6
![24](https://graph.baidu.com/resource/11199ae1110e771fe5d3d01565678407.jpg)


* label s5：store5、store8
![25](https://graph.baidu.com/resource/111363e57e33dbe49e7cf01565678431.jpg)



* label s6：store 1、store4、store6
![26](https://graph.baidu.com/resource/111120d483b504ca9b54f01565678452.jpg)


* 添加scheduler前：region65的peer的label都属于  
label s6：（store 1、store 4、store 6）
![27](https://graph.baidu.com/resource/111069f3b890b816c989f01565678477.jpg)


* 添加./pd-ctl scheduler t_add transfer-regions-of-label-to-label-scheduler zone s6 zone s5，执行成功
![28](https://graph.baidu.com/resource/11104ce773c72c5d78a7e01565678501.jpg)


* 后续查看region65
![29](https://graph.baidu.com/resource/111a13373493389d7287101565678524.jpg)


* 添加scheduler后：region65的peer所属的label分别是：   
label s5：（store 5、store 8）       
label s6：（store 1）
**之前label s5下没有region65的副本，且label s5下有两个store，因此transfer-regions-of-label-to-label-scheduler zone s6 zone s5会将label s6下面的region65的两个peer都移动到label s5下。**
## 4. transfer-regions-of-keyrange-to-label-scheduler
### 作用：
* 输入start key值以及start key之后几个range，并将其转移到指定的label下
### 用法：
![16](https://graph.baidu.com/resource/111fa39be3fde3322fccb01565678167.jpg)


### 示例：
* region42如图所示：
![17](https://graph.baidu.com/resource/111507df9454966201f3b01565701968.jpg)


* 添加scheduler成功，其中limit为从start key开始后的15个region。
*  由于./pd-ctl scheduler t_add transfer-regions-of-keyrange-to-label-scheduler 的执行，因此直接导致添加了很多迁移到s8的scheduler。

![18](https://graph.baidu.com/resource/11119a1855286c5af1e4a01565701986.jpg)


* 转移后的region42
![19](https://graph.baidu.com/resource/111c4ed369932bf5a3d7c01565701876.jpg)





## 5. transfer-table-to-label
* 转移前的region14：
![21](https://graph.baidu.com/resource/111e016976827a56689d201565702181.jpg)



* 添加scheduler：./pd-ctl scheduler t_add transfer-table-to-label mysql user zone s8后，其中user是系统表，现在只添加了一个scheduler到scheduler列表中，是因为当前user表还没有数据，因此只在一个region中。
![22](https://graph.baidu.com/resource/11166908b458a098e2b7801565702232.jpg)


* 转移后的region14
![23](https://graph.baidu.com/resource/11163e63bc8ac60332b1901565702213.jpg)







