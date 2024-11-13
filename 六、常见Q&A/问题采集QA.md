## 1、技术问题
### 1.0、架构基础问题
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1285871/1687249104519-3786ffc2-9faf-45bc-aa23-e0c8b2dae33d.png#averageHue=%23faf9f7&clientId=ud0aa72f5-02b8-4&id=AVm3h&originHeight=1252&originWidth=2326&originalType=binary&ratio=1&rotation=0&showTitle=false&size=342087&status=done&style=none&taskId=ub414d82d-88ec-4d9d-a37b-d330bc1e856&title=)
austin项目**核心流程**：austin-api接收到发送消息请求，直接将请求进MQ。austin-handler消费MQ消息后由各类消息的Handler进行发送处理

---

**Q ：为什么发个消息需要MQ？**
**A：**发送消息实际上是调用各个服务提供的API，假设某消息的服务超时，austin-api如果是直接调用服务，那存在**超时**风险，拖垮整个接口性能。MQ在这是为了做异步和解耦，并且在一定程度上抗住业务流量。

---

**Q：能简单说下接入层做了什么事吗？**
**A：**
![](https://cdn.nlark.com/yuque/0/2022/jpeg/1285871/1648816455463-bcf3ef2e-5ec4-452e-b8a2-5a263474d9d5.jpeg#averageHue=%23f5f2ec&clientId=ue57b3c3d-a2a2-4&from=paste&id=u359060ac&originHeight=450&originWidth=2488&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=ue8d0e4a2-6803-4784-9acc-7ef1dbfc833&title=)

---

**Q：austin-stream和austin-datahouse的作用？**
**A：**austin-handler在发送消息的过程中会做些**通用业务处理**以及**发送消息**，这个过程会产生大量的日志数据。日志数据会被收集至MQ，由austin-stream流式处理模块进行消费并最后将数据写入至austin-datahouse

![](https://cdn.nlark.com/yuque/0/2022/jpeg/1285871/1648816455469-ea061a57-7948-415c-b930-5f368703a228.jpeg#averageHue=%23f6f3f0&clientId=ue57b3c3d-a2a2-4&from=paste&id=u332cf364&originHeight=942&originWidth=2582&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=ubbd135b7-a3c5-4f55-8ab2-6a210a07c51&title=)

---

**Q：austin-admin和austin-web和austin-cron的作用？**
**A：**autsin-admin是austin项目的前端项目，可通过它实现对管理消息以及查看消息下发的情况，而austin-web则是提供相关的接口给到austin-admin进行调用（austin项目是前后端分离的）
业务方可操作austin-admin管理后台调用austin-web创建**定时**发送消息，austin-cron就承载着定时任务处理的工作

---

### 1.1、发送接口
**Q**：**为什么消息的接收者不能大于100个？ 是austin本身项目提供不了那么多的服务还是说别的考虑因素？**
**A：**总需要有限制，不然传入几W个也不合适。很多渠道支持也就100多，100个限制挺合适的。

---

**Q：**![image.png](https://cdn.nlark.com/yuque/0/2022/png/1285871/1662298955390-4060502a-7495-4c3e-80b7-fbdaf0655de4.png#averageHue=%23e9e9e9&clientId=u9a9795ed-fbd4-4&from=paste&height=86&id=U8I53&originHeight=172&originWidth=1246&originalType=binary&ratio=1&rotation=0&showTitle=false&size=96383&status=done&style=none&taskId=u6694f58b-2282-48bd-8262-1a0c7a71d4c&title=&width=623)
**A：**第一个问题：代码中已经限制了一次接收者**只能最多100个**（100个已经够了，下发渠道一般都会做限制，所以在消息推送平台直接限100个），避免内存OOM这种情况。第二个问题：接收者有千万个，一次请求能组装进100个接收者，量级会下降，**其实并不是什么大问题。**

---

**Q：send和batchSend接口有什么区别？**
**A：**send接口是 相同的文案 ，最多发给100个人，batchSend接口是支持**批量发送不同的消息给不同的人群**。

---

### 1.2、接入层责任链实现
**Q**：**责任链中的两部检查，就是在查数据库前后的两次检查，能不能合在一起  你这边是分的入参检查 + 业务检查（比如说这个业务检查会检查手机号或者邮箱格式等，这个是不是入参检查的时候也可以做，或者说这么设计的目的性？）**
**A：**前置检查的是入参的，后置检查的是有了模板信息后的（没有得到模板信息，那就不知道当前的请求发送渠道，当前的ID类型是什么，而模板信息是在组装参数里才得到的）。

---

**Q：为什么责任链 processModel 要定义为 泛型？**
**A：**因为使用泛型**能避免强转**，这个看pull request 的代码就懂了。[https://gitee.com/zhongfucheng/austin/pulls/16/files](https://gitee.com/zhongfucheng/austin/pulls/16/files)

---

**Q：假如说现在有同一个模板，就是一个业务方，用一个模板给10个不同的用户发短信，但是我看代码假如每次都调用send接口，那么中间都会打一次mysql，这里会不会有问题？比如说需不需要缓存，因为这种情况下每次请求mysql拿到的都是同一个模板数据，那么能不能作为热点数据缓存起来 比如说是用redis 或者说你这里业务实现时候是怎么考虑的呢？ **
**A：**是否缓存在文章里应该也有提及？如果性能确实跟不上了，那是可以缓存的。但是像这个业务场景下，缓存的意义并不大（毕竟你引入了缓存，就会有一致性的问题，这时候就需要做取舍了），**通过主键请求mysql，查询速度很快的**。

---

### 1.3、线程池核心参数设置
**Q：为什么设置线程数的时候core=max=2呢，首先为什么是相等呢？  **
**A：**为什么相等，这个跟线程池的原理有关系。如果workQueue阻塞队列已经满了，则判断当前线程数是否大于maximumPoolSize，如果没大于则**创建新的线程**执行任务。曾经我在线上环境下看到过核心线程数为10，最大线程数为1000，由于线程池处理速度过慢，导致线程数一直保持1000，严重影响到了性能。

---

**Q：其次线程数2是基于什么考虑的 是任务类型嘛（cpu/IO？）？**
**A：**这个初定的值没这么多讲究，看着定的，如果真不够用了，后面再改。参数这东西很难说一次就能直接定对的，能有很科学的讲究的。

---

**Q：那个core=max=2为什么就效率高了，这样做不就线程池内永远只有两个线程了吗？那1000个线程为什么就慢了，1000个线程不是更快了吗？  意思并发执行这里要考虑线程切换的开销吗？   那线上环境有没有可能这两个值不一样还是说业界习惯这么做两个值设置成一样**
**A：**确实是需要考虑线程切换的问题，并不是线程越多，就会越快。core=max=2并不是说它效率就一定高，具体值是合理的值多少是需要调试出来，只是**相等我们更容易**把资源控制在一定的范围内（不恰当地可以类比JVM设置最大小堆，一般也设置为一样）。线上当然是有可能设置两个值不一样，不过**我的习惯是设置一样的**。

---

**Q：阻塞队列为什么用VariableLinkedBlockingQueue？  容量128是基于什么考虑的呢？**
**A：**这个具体得看开源项目动态线程池了，我印象中是有文章讲过的。阻塞队列只有VariableLinkedBlockingQueue类型可以修改capacity，该类型功 能和LinkedBlockingQueue相似，只是capacity不是final类型，可以修改， **VariableLinkedBlockingQueue参考RabbitMq的实现。**
参考动态线程池文章：[https://juejin.cn/column/7053801521502224392](https://juejin.cn/column/7053801521502224392)

---

**Q：丢弃消息夜间屏蔽去重服务（重点）为什么不能放在接入层做呢？放在消费侧做的话是不是太占用线程池资源了呢、就是我明明已经经过MQ了，又进了线程池了，结果处理的时候才发现应该去重掉、 这样是不是太麻烦了？**
**A：**主要是想要让**接入层更加轻量级，业务方侧调用时能很快地就得到响应**。把丢弃消息夜间屏蔽去重服务如果放到接入层，那会一定程序上影响到响应时间。如果接入层把这些业务逻辑都干了，那实际上消费侧就没什么逻辑了，消费侧轻量级也不会带来什么好处。

---

**Q：为什么有的地方设置允许核心线程超时，有的不允许？**
**A：**因为允许线程超时（allowCoreTimeOut = true）这种场景，我是想线程池如果没用到了，就能释放资源的。而（allowCoreTimeOut = false），我的业务场景就需要这些线程池去处理，属于常驻资源的

---

### 1.4、优雅关闭线程池
**Q：**这里为什么要调用awaitTermination ？只调用shutdown关闭行不行。调用完awaitTermination 后 有可能有任务没有执行完而终止了  需要紧接着调用一下shutdownNow把未执行完的任务获得一下吗 ？如果有任务未执行完那该怎么办呢？
![image.png](https://cdn.nlark.com/yuque/0/2022/png/1285871/1662299199154-b4f56489-601a-41c9-8836-2b01c0d99c68.png#averageHue=%232e2c2b&clientId=u9a9795ed-fbd4-4&from=paste&height=320&id=cUTh9&originHeight=640&originWidth=1192&originalType=binary&ratio=1&rotation=0&showTitle=false&size=189486&status=done&style=none&taskId=ud3c63e21-4cea-4960-8f79-2f439979702&title=&width=596)
**A：**只调用shutdown方法会一直等待（因为shutdown方法本身就是会等任务处理完才关闭），而awaitTermination就是为了**定一个超时时间**，过时了就不等了，未执行完的任务也强制干掉了(直接中断)。**优雅关闭**主要的意思是：**任务不能立马关闭掉，但是我可以设置一个超时时间，这个超过这个超时时间了，关闭了业务是能接受的**。

---

### 1.5、去重实现lua逻辑
**Q：limit.lua的逻辑？为什么要移除时间窗口的之前的数据？为什么ARGV[4]参数要唯一？为什么要expire？**
![image.png](https://cdn.nlark.com/yuque/0/2022/png/1285871/1662301599423-05b3465c-6641-4afb-94ea-8e20796c96f9.png#averageHue=%232e2e2e&clientId=u9a9795ed-fbd4-4&from=paste&height=539&id=lRMUi&originHeight=1078&originWidth=1430&originalType=binary&ratio=1&rotation=0&showTitle=false&size=198666&status=done&style=none&taskId=ubf71ad13-4af8-41b9-a5be-f04ad1c5ff9&title=&width=715)
**A：**使用**滑动窗口**来保证N分钟达到N次进行去重。滑动窗口可以回顾下TCP的，也可以回顾下刷LeetCode时的一些题，那这为什么要移除，就不陌生了。为什么ARGV[4]要唯一，具体可以看看zadd这条命令，我们只需要保证每次add进窗口内的成员是唯一的，那么就**不会触发有更新的操作（我认为这样设计会更加简单些）。**而唯一Key用雪花算法比较方便。为什么expire？，如果这个key只被调用一次。那就很有可能在内存常驻了，expire能避免这种情况。
PS：该代码的由来，可看这次的pull  request：[https://gitee.com/zhongfucheng/austin/pulls/19](https://gitee.com/zhongfucheng/austin/pulls/19)。

---

### 1.6、发送记录落盘和性能
**Q：现在什么渠道会记录发送记录？MySQL会不会撑不住？是否需要分库分表**
**A：**现在只有短信是有把记录存储在MySQL，其他渠道都是记录日志（线上环境，最后会落到hive里）。最主要的是短信这渠道比较重要，因为还会涉及到短信对账，所以会**落库**。由于消息推送下发，短信的下发的记录量肯定会越来越多的。
之前我们线上环境大概有1亿4的量，我们没有做分库分表（这一点主要是最开始没考虑进去吧）。但是，其实一亿多的量，在数据库建好索引，我们后台其实只有**一天时间+手机号**查的功能，**是够用的**。另外，因为我们hive里会备份历史的数据，所以我们其实可以**删除MySQL的历史数据**来有更好的查询性能。
当然了，如果公司里有条件，最好是分库分表啦。（就单单短信业务下发记录而言，没有分库分表也是可以的）

---

**Q：我看发送短信以后，直接把resopnse进行组装插入到sms_record里面了，但是还有个拉取回执的接口，是通过.setSmsSdkAppId 直接将拉取的结果ms_record也存到这个表里面了而不是去更新，这里是为啥呢**
**A：**总体来说，这样设计会**简单点**(insert 比update简单），**灵活点**(面对不同的渠道都可以)。
1、如果是更新(update)的话，那可能需要设计更多的字段去把相关的信息写到一行记录
2、有的渠道可能是按回执的条数计费，而接口没有直接返回计费条数的字段

---

### 1.7、定时任务逻辑
**Q：定时的消息任务，不是要经由taskHandler去处理吗，实现类里面生成了一个多例实例“crowdBatchTaskPending”对象。是不是一个csv文件就对应一个“crowdBatchTaskPending”对象，整个csv文件不管有几千行row，都是一个“crowdBatchTaskPending”实例来处理的啊？ 那rowInfo为啥要放入到linkBockQueen队列中，再“crowdBatchTaskPending”实例中，用单线程转换到成员变量tasks这个集合中啊？为啥不直接开始就用tasks集合接受rowInfo对象，针对这个List来循环判断是否满足size，再转给batch发送方法啊**
**A：**确实是一个csv文件就对应一个“crowdBatchTaskPending”对象。至于为什么要放到linkBlockQueue，其实这个场景下，这个的功能所实现代码是差不多的。假设让你去实现，也差不多类似的代码。**AbstractLazyPending  这个你可以理解为一个组件**，只要走批量的都可以用这个组件，而 CrowdBatchTaskPending 只是其中的一块业务而已。这样会更加通用些。

---

### 1.8、业务指标监控
**Q：监控是怎么实现的**
**A：**监控是**Graylog**解析日志，依赖自带的dashboard就行了（你公司用什么，你就接什么就好了，比如用的是ELK，那就在ELK上做）。操作步骤：1、在**input里增加Extractor**，（因为是json格式，所以增加json的Extractor就好了）2、增加完Extractor后，可以直接在**fields里找到 RT字段**，做统计或者图表就行了。
![05620272dc4a37c0d1a54ee6cb6da32.jpg](https://cdn.nlark.com/yuque/0/2023/jpeg/1285871/1679627170934-9d5842d6-07f2-40bf-afcc-2b8b33ca9a76.jpeg#averageHue=%23f8f7f7&clientId=u9feb09e3-f9ac-4&from=paste&height=542&id=u86ce2c53&originHeight=542&originWidth=1912&originalType=binary&ratio=1&rotation=0&showTitle=false&size=81534&status=done&style=none&taskId=u342b1ea3-e02f-4574-af8b-b71464d5b28&title=&width=1912)
![03fb2c658b0d996c612ed35422c5766.jpg](https://cdn.nlark.com/yuque/0/2023/jpeg/1285871/1679627181026-7cb55805-ea7e-4001-a7a8-5996d56e72bc.jpeg#averageHue=%23f5f4f4&clientId=u9feb09e3-f9ac-4&from=paste&height=639&id=fBieY&originHeight=639&originWidth=328&originalType=binary&ratio=1&rotation=0&showTitle=false&size=30119&status=done&style=none&taskId=u5f4c95cf-c60c-4a72-9e3c-6b3fbc3eec3&title=&width=328)![e9824368bd863583d8090a42e342a98.jpg](https://cdn.nlark.com/yuque/0/2023/jpeg/1285871/1679627162491-638b2276-ab67-4bab-9f92-b06fc8716a46.jpeg#averageHue=%23f7f7f7&clientId=u9feb09e3-f9ac-4&from=paste&height=328&id=fo9pI&originHeight=663&originWidth=762&originalType=binary&ratio=1&rotation=0&showTitle=false&size=37413&status=done&style=none&taskId=u2a6134d5-3ada-49e9-9182-d4df68ac9e6&title=&width=377)

---

### 1.9、数据链路追踪逻辑
**Q：如果没有stream，数据清洗的话，只有通过看日志的方式才能知道消息下发的情况了，是吧？我是想知道链路追踪这个功能，还有其他的实现方式吗**
**A：**是的，没有stream模块数据清洗，只能通过日志去看了（短信例外，可以通过入库信息看）。链路追踪的功能，**所有流式处理平台**都可以实现的，或者**不用流处理，直接消费MQ的数据也能清洗出来**。本质上就是去拉去日志，然后将拉去的日志按照规则过滤下，然后把我们关心的数据放到数据库（redis、MySQL等）。其实就是**看消息发送的状态，更直观化**。

---

**Q：邮件状态中，用户点击状态Click是不是还没写？如果实现的话，怎么确定用户点击了呢**
**A：**是的，clilck点计状态这块目前是没有实现的。一般在电商的系统里，click点击这种事件会有一个的表来**全站的点击（也会衍生出一个实时的点击消息topic）。**而在austin里要做的就是，在austin-stream实时流处理模块里，消费上面提到的**点击topic**，我们可以过滤会这个topic消息里带有**消息推送的参数的**点击（com.java3y.austin.support.utils.TaskInfoUtils#generateUrl），那清洗到现有的维度上就好了。
至于怎么确定用户点击了，这个很简单：url是一个链接，点击相当于访问了这个链接，链接呗访问，那不就是说明被点击了吗。这时候，前端会把这次**用户访问链接**的上报给数仓（采集），那最后就生成前面提到的（**实时的点击消息topic）**供公司各个业务团队使用。

### 1.10、access_token找不到
**Q：想问一下我在模板消息测试钉钉发送消息的时候，前端报操作成功，但是后台提示没有access_token，我想问一下咱这个项目的token_access在哪里设置呢？**
**A：**可以手动调用 com.java3y.austin.web.controller.RefreshTokenController#refresh 这个接口，正常是在xxl-job定时任务里刷新该token的，对应的任务位置：com.java3y.austin.cron.handler.**RefreshDingDingAccessTokenHandler**#execute 和com.java3y.austin.cron.handler.**RefreshGeTuiAccessTokenHandler**#execute
如果你已经部署了xxl-job，建议是把这两个任务在xxl-job后台**手动**创建出来。

---

### 1.11、前端使用
**Q：调用参数参数可以无限加**
![75ea6c20dfb251406bb72b5265e6908.png](https://cdn.nlark.com/yuque/0/2023/png/1285871/1679628003815-1db0b9f3-ece2-4f75-8810-1633bd996902.png#averageHue=%23fefdfd&clientId=u9feb09e3-f9ac-4&from=paste&height=312&id=u43f1d39f&originHeight=528&originWidth=624&originalType=binary&ratio=1&rotation=0&showTitle=false&size=12783&status=done&style=none&taskId=ua5601493-7938-41cc-a74d-d422b6f50f5&title=&width=369)
**Q：修改不了上传的人群文件**
**Q：前端太丑不好用**
**A：**前端用的是amis低代码平台，现在实现只能说是**可用，**达不到**好用**的。如果觉得页面有问题，那就是我没实现。

---

### 1.12、项目环境参数
**Q：application.properties的项目环境参数是怎么指定的？比如${austin.database.port:3306}**
**A：**一般情况我们上测试或者生产是要修改的配置的，如果这样写就不用每次提交代码的时候配置了，可以把这些参数配置**在系统**里（如K8S平台会有**ConfigMap**配置）。${SENTINEL_MASTER:poyee-master}，先读 ${SENTINEL_MASTER}变量，如果为空就用后面的默认值。

---

### 1.13、为什么Task对象是多例且要给Spring管理
**Q：这个Task对象为什么要加入Spring容器啊，直接new不行吗？为什么定义为SCOPE_PROTOTYPE（多例）？**
![c68b63a62b4c839a6f3d516222c2848.png](https://cdn.nlark.com/yuque/0/2023/png/1285871/1682059189101-c48d088b-2465-4b3c-8ae3-703d6bf224f2.png#averageHue=%232d2c2c&clientId=ub839bbf8-d494-4&from=paste&height=384&id=uea39fd05&originHeight=384&originWidth=1504&originalType=binary&ratio=1&rotation=0&showTitle=false&size=32656&status=done&style=none&taskId=u34c90249-6fb7-467f-8ffa-40f450ed1a9&title=&width=1504)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1285871/1682059264048-9ac8cef9-4d5a-404d-9706-b170ca113e0e.png#averageHue=%23342c2a&clientId=ub839bbf8-d494-4&from=paste&height=180&id=ub2c75b4e&originHeight=180&originWidth=577&originalType=binary&ratio=1&rotation=0&showTitle=false&size=20048&status=done&style=none&taskId=u601e93a9-632a-4c00-bf4e-f6b05515195&title=&width=577)
**A：**这个Task对象里边会有**Spring相关的依赖对象**，如HandlerHolder/DeduplicationRuleService对象等等，那Task对象想要**正常使用**，只能也加入到Spring环境（容器）。。为什么是多例，因为**TaskInfo对象**是多例（在多线程环境下，如果**只有一个Task对象，会有线程安全的问题**）

---

### 1.14、保证高可用
**Q：项目是怎么保证高可用的？**
**A：**高可用更多的是**运维**层面的，比如我部署了多台机器，即便有一台挂了，也不影响我服务正常使用。比如上了K8S，我在重启发布的时候，会先拉去新的机器，起来之后再把老的下线掉。**开发更多的是配合运维**去做一些服务的监控（如果服务挂了，能第一时间告警响应出来）。而项目应用层面上的，我们也就多打些日志，依赖的中间件如果挂了，能及时告警（**监控+告警**）

---

### 1.15、RocketMQ动态多group
**Q：我看kafka实现是通过 KafkaListenerAnnotationBeanPostProcessor 实现 动态的多group的，在RocketMq有这个实现吗？**
**A：**本来没有的，后来已经有PR提到了rocket-spring上了（已被merge）：

- [https://github.com/apache/rocketmq-spring/pull/480/commits](https://github.com/apache/rocketmq-spring/pull/480/commits)
- [https://github.com/apache/rocketmq-spring/issues/491](https://github.com/apache/rocketmq-spring/issues/491)

---

### 1.16、为什么使用dynamicTP
**Q：为什么使用dynamicTP，有没有对比其他的方案**
**A：**动态线程池现有比较出名的轮子都来源美团的那篇文章[https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)。 [Hippo4J](https://github.com/acmenlt/dynamic-threadpool) 和[dynamic-tp](https://github.com/lyh200/dynamic-tp) 当时都是在我考虑范围之内的（一般就看知名/社区活跃/近期是否有更新维护）。用到的技术栈都是大同小异，没有本质区别（**理论都是基于美团的文章**），而我选择[dynamic-tp](https://github.com/lyh200/dynamic-tp) 就当时因为它可以**编程式**去把普通线程池包装成动态线程池以及获取 （不过，这个 [Hippo4J](https://github.com/acmenlt/dynamic-threadpool) 后来也支持啦，[Hippo4J](https://github.com/acmenlt/dynamic-threadpool)的功能更全，其实可以选这个去接入）。

---

### 1.17、redis 需不需要考虑持久化
**Q：redis 需不需要考虑持久化**
**A：**一般在公司里边，我们这些使用redis的，其实是**比较少去关注它的持久化方式（**是使用AOF还是RDB，还是说混合AOF/RDB)。基建比较好的公司，会在你接入redis的时候，新建redis实例，而这时候会让你勾选使用场景，就是会让你通过不同的应用场景进行配置选择，比如说，业务上是允许重启时部分数据丢失的，那RDB就够用了。对于austin来说，是非强依赖redis的，用哪种持久化方式也并那么看中。

---

### 1.18、什么是离线和在线数据？
**Q：面试官问我，哪些数据是实时的，哪些是离线的，为什么要这样分出来。我看面经说明里面写“写到hive里的叫离线数据（很少会提供在线接口服务并且数据落到hive需要一定时间，而写到redis则会提供在线的接口服务查询（这个就叫做实时数据）”。**
**Q：面试官不满意这个答案，问既然都存到hive，为什么不直接去查？**
**A：**查hive也没这么快啊，再说了，hive里的是明细数据，要查出 用户/模板的维度结果，还得做一层聚合。

**Q：redis的数据保留多长时间？多久的数据算实时数据？**
**A：**模板维度7天，用户维度1天（这个代码里有）

**Q：kafka存储的日志数据，flink消费之后写到redis和hive，他们之间有多少的时间差？**
**A：**flink 落到hive就看策略，有的公司可能5min，有的30min。一般是大数据那边配置的，其实就是多久消费一次kafka的数据，写入到数据仓库。在后台的功能上或者提供出去的在线接口，很少说会直接查hive的，因为**查得慢**！。**数据仓库是给分析用的，不是提供在线服务的**。
redis的数据是给到消息管理后台去查的，下发了以后就能直接查到，而落到hive一般是有延迟的，一般是T+1做报表的

---

## 2、业务问题
### 2.1、限流
**Q：8000人/QPS是什么意思呢 就是每秒请求是8000嘛？腾讯云短信3000qps，意思说是说sms下发渠道每秒下发3000条短信呗？**
![image.png](https://cdn.nlark.com/yuque/0/2022/png/1285871/1662297681647-2fd03347-dec6-43c3-84f3-0d08f42b1bd9.png#averageHue=%23d8d8d0&clientId=udfb7d369-b1fe-4&from=paste&height=118&id=XBw54&originHeight=236&originWidth=1218&originalType=binary&ratio=1&rotation=0&showTitle=false&size=354239&status=done&style=none&taskId=ufd5950b9-8043-42d4-b581-ac77e25f231&title=&width=609)
**A：**个推我们这边限制的是**下发人数**，每秒能下发8000人。腾讯云渠道限制的是**调用次数3000**，但是一次可以下发给多个用户。为什么是不同的呢，主要**跟具体的渠道**有关系。腾讯云是官方的限制，这个是没得说的。而个推PUSH我们是根据**系统处理能力以及业务能接受的程度定的一个规则限制**。

---

**Q：**
![image.png](https://cdn.nlark.com/yuque/0/2022/png/1285871/1664806136287-06695175-20cd-4d34-bfdc-4a93453ffcdd.png#averageHue=%23bacfe2&clientId=ue2928c67-a007-4&from=paste&height=678&id=oA4zV&originHeight=1356&originWidth=1782&originalType=binary&ratio=1&rotation=0&showTitle=false&size=339459&status=done&style=none&taskId=uf3127e8b-117c-451d-8a92-b784df73d9a&title=&width=891)
**A：**第一个问题理解是对的，就是**按照发送的用户量限制调用下游渠道接口的频次**。第二个问题，因为Email比较显著，对于腾讯邮件确实渠道是需要限流的，所以在此实现了个案例。其实对于腾讯云短信，还有钉钉渠道之类的，也需要实现（在初始化Handler把限流参数传入即可）。第三个问题，**3这个限流值只是做参考**，实际按自己的服务器数量来调配（可以通过分布式配置中心调整）。**实际项目的邮件qps确实就那么少**（对于腾讯邮件来说给的量级的确不大，其他邮件没考究过）

---

### 2.2、业务使用
**Q：业务方发送定时任务的话，是不是就不用走前端系统了，毕竟前端系统这是给运营用的，业务方能不能直接对接austin-cron模块**
![image.png](https://cdn.nlark.com/yuque/0/2022/png/1285871/1664285384401-57cba917-3aa1-42aa-aa25-9bc74bf28af7.png#averageHue=%23f8f6f4&clientId=u2b28710c-b8f2-4&from=paste&height=414&id=u90475499&originHeight=827&originWidth=1435&originalType=binary&ratio=1&rotation=0&showTitle=false&size=203393&status=done&style=none&taskId=u383eb8b6-5755-48ed-a455-26cb921495d&title=&width=717.5)
**A：**不能。首先明确一点，只要是使用消息推送平台的用户，就是我们的业务方。业务方的大致可以分为几个角色：运营、技术、客服。其次，技术业务方要发送定时任务的消息，**如果能通过消息推送后台定时发，他们是绝对不会写代码自己调用的**。再而，如果技术侧业务方要实现定时推送的功能，在这个场景下，需要发送的文案基本都是要跟发送者关联的（**需实时获取的**），定时模块是做不了这样的功能的。

---

**Q：如果技术业务方需要发送定时任务消息，通常这个消息模板需要填充一些业务相关的参数，austin是不是不支持这种需求？能否像实时消息那样，在业务端定时调用austin的发送消息接口实现这种功能？**
**A：**已经支持的，例如下面的csv文件，定义了content和url列，那在模板填写的时候，只要有{$content}和{$url}写在文案上，那就会解析
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1285871/1677594772585-97812f35-15b5-4b32-a30e-f1513b8216e5.png#averageHue=%23393b3d&clientId=ua28e4e33-530b-4&from=paste&height=93&id=V7kwI&originHeight=186&originWidth=1342&originalType=binary&ratio=2&rotation=0&showTitle=false&size=35838&status=done&style=none&taskId=u4dc7a864-f94c-4498-adb9-8e907d18d2e&title=&width=671)

---

**Q：template列表上的复制功能是干嘛使的?**
**A：提高运营效率**：我们当时是运营每天都会发push消息，而模板每天的配置都是一样的（除文案），复制就为了给他们去建新的模板（他们只要改下文案和标题，不用重新填其他的信息）

---

**Q：作为web侧调用。必须得现有template，然后再基于template构建实际的渠道消息。但是作为api调用方的话，不可能先创建template再调用吧。这块sendRequest是必填的，调用侧也应该走这个入口呀。api调用也得创建template的话，那这样，每次加一个template，调用侧不会涉及到改代码吗？**
**A：**用平台，肯定要有平台的约束。**消息推送就是得有模板才能下发**，不然出了问题都不好查。**新的业务场景要加新的template**，那新的场景api调用侧**本身就要新写代码逻辑**支持。

---

### 2.3、短信
**Q：短信没收到呀，也没报错。**
**A：**比如你申请的短信模板是 **验证码为 $1，您正在登录，若非本人操作，**这时候你在消息推送平台里只用填写**$1 **的文案，**不要把整个申请下来的模板复制进去**。如果日志显示发送成功，那非常有可能就是**文案**的或者**手机号**本身的问题（也有可能短信账号已经没有余额了）。这时候你可以直接去**数据库表（sms_record** ）里查记录（如果显示没有问题，那只能看回执了，有问题会记录在库中的）

---

**Q：腾讯云短信下存在多个模板，如何指定模板发送短信？另外短信模板在多变量下，发送短信会失败，看下了代码，好像只支持单变量。**
![image.png](https://cdn.nlark.com/yuque/0/2022/png/1285871/1671534810500-d2f4f97c-a82d-401b-a1fb-9e25e7daeec7.png#averageHue=%23302e2d&clientId=u6b8352ee-3101-4&from=paste&height=175&id=ub7db7a32&originHeight=349&originWidth=970&originalType=binary&ratio=1&rotation=0&showTitle=false&size=64085&status=done&style=none&taskId=ue6437ae2-bff5-4375-b761-c0499ad17b6&title=&width=485)
**A：**接入短信这块主要是做了个演示（需要根据实际情况自己改动）像到公司生产级别的，发送通知类、营销类就**可以没有短信模板**的（**去商务**）。当然了，这块也可以改造下，**类似于微信模板消息一样，维护一个短信模板列表**。

---

**Q：腾讯云短信回执拉取不到**
**A：**1、需要找**客服开通权限**才能拉到记录 2、把com.java3y.austin.handler.receipt.MessageReceipt#init的注释给打开

---

**Q：我看这里拉取短信回执是另起的单线程。2秒执行一次(或者时间更短)，如果在拉取回调时时候短信提供方接口挂了，或者迟迟没有返回，线程阻塞在那里，这个有没有可能会造成cpu资源浪费;或者这可能会导致服务响应时间变长，因为该线程没有被释放，服务不能为其他请求提供响应;对发消息的主流程造成影响呢;我看在真正调用pull接口时候，还会创作一些对象，当线程被hold住时，这些对象可能会一直占用内存。如果线程持续hold住，这些对象可能会在服务中引起内存泄漏。**
**A：**考虑真细啊！哈哈哈。现在计算机基本都是多核的，一个线程是小问题啦，况且线程等待网络io时，cpu是可以干其他事的。所以，一个线程被网络io阻塞了，是不会影响很大的。**其他服务不是该线程处理的，所以自然也不会受影响**。**至于cpu系统资源嘛，不用过于担心**。就比如我们从kafka拉取数据，实际上它也是有一个线程while(true){}去拉取数据的。至于线程hold住，我们拉去表面是使用java代码，但内部一定是网络交互的（那网络交互这个连接一般都会有设置**超时时间**）。可以点进去sdk里的代码看看，内部是通过http访问的，设置了超时时间的。只要不是死锁，**还谈不上内存泄漏**的，就会有释放的时候，线程释放内存空间后面也会释放。

---

**Q：拉取回执消息，用的是单线程池，循环调用pull接口的实现类去拉取，如果腾讯渠道拉取回执报错以后，那么云片pull是不是就会不执行呢**
**A：**不会哈，比如腾讯pull的代码里头会有try catch，即便腾讯云拉取失败了，也不会影响到云片渠道的。
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1285871/1682686804695-0915e79b-83a2-4c78-915a-8f967ed61011.png#averageHue=%23232427&clientId=u7e7009a8-ba89-4&from=paste&height=507&id=u81c2369c&originHeight=1014&originWidth=2558&originalType=binary&ratio=2&rotation=0&showTitle=false&size=201595&status=done&style=none&taskId=uec9d9a0d-73e6-4616-b966-a18198c5976&title=&width=1279)
### 2.4、接口安全
**Q：Austin对外暴露的消息发送接口api，如何保证安全呀（有防止接口恶意调用的实现吗）**
**A：**1、austin在公司里，一般是作为一个**内网的服务**提供出去，不会到外网里。2、假设要提供外网服务，一般是通过【开放平台】包一层出去，安全是开放平台的事了。开放平台里就会有类似有这些功能（签名 加密 IP白名单 鉴权 参数校验）

---

### 2.5、站内信（IM）
**Q：什么是IM（站内信）**
**A：**IM是即时通信，可以是「**官方账号**」推送通知/营销消息，可以是用户之间的「**单聊**」，可以是用户之间的「**群聊**」，甚至客户端内你能看到的**弹窗**（这也可能是IM技术）
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1285871/1678580244438-8b997184-74bd-4a82-9817-c0b60d3caae6.png#averageHue=%23f7f8f0&clientId=u06b270b9-b2d5-4&from=paste&height=265&id=u36b87452&originHeight=874&originWidth=1080&originalType=binary&ratio=2&rotation=0&showTitle=false&size=557184&status=done&style=none&taskId=ude7ad88a-2c55-498e-902d-cc3694c5d9a&title=&width=328)![image.png](https://cdn.nlark.com/yuque/0/2023/png/1285871/1678580525256-c984b11f-9654-407d-806b-5a9b986890d2.png#averageHue=%23f7f6f5&clientId=u06b270b9-b2d5-4&from=paste&height=370&id=ICryp&originHeight=1767&originWidth=1170&originalType=binary&ratio=2&rotation=0&showTitle=false&size=466846&status=done&style=none&taskId=u5bfc8ad8-21bc-4c43-901d-227eba1b8b3&title=&width=245)
IM是一项很复杂的技术，一般由**另一个系统组成**，而适合消息推送平台对接的有两种：**「官方账号」推送消息**，**客户端弹窗**。
以前我做的系统确实对接了这两种消息类型（本质来说就是IM），但现在由于没有客户端环境，而IM现在大多数都是需要去购买现成的服务，才没有走对接
（PS：MOGU本来是自研的，后面由于**维护成本**，最后还是选择了购买云服务）。

---

### 2.6、发送失败重试
**Q：比如有一条付款成功消息，会有多个渠道都会发送。此时调用多次接口吗？有没有考虑一个渠道，比如钉钉，第一次失败了。考虑重试3次，还是不行，再更换其他渠道?**
**A：**主要是，失败大概率都是接**收者本身的问题**（很低可能是接口调用的问题），而发送的消息往往都是**异步通知**的（调用接口成功，但还要**接收回调**才知道是不是**真发到用户上**了），push和钉钉和短信等渠道我印象中都是异步的，要**拉取回执**的。我们只有**短信这种渠道**才要做这种对“**接口失败容灾**”的处理（毕竟是最重要的渠道）。

---

### 2.7、撤回消息
**Q：消息撤回这块没有文档写呀。看实现也就一个钉钉实现了。也主要是依赖template？**
**A：**是的，目前撤回依赖template，其他渠道**要么没有撤回的api，要么就是我懒得实现**

---

**Q：我感觉template下不是会有消息吗？为什么不是发送消息的回执ID这种**
**A：**现在设计简单些，要做**精细化撤回**，肯定是按具体的消息id的（相对复杂和麻烦点）
更新：已经可按照消息id进行撤回（供接口调用，前台无法细化到消息维度）

---

**Q：按照模板ID，撤回是不也会有时间限制。早发送是的消息就撤回不了了。第三方也有限制吧。那撤回失败还得记录状态咯**
**A：**目前撤回的信息现在是存redis里的（有过期时间），确实第三方也是有限制。如果撤回失败，**做简单点发个通知就算了**（发现过期了，就说撤回失败了）

---

### 2.8、指定发送者ID
**Q：发送渠道就可以决定接受者id类型了，为什么还会有接受者id类型这个选项？**
**A：**嗯，是这样的。正常来说，当业务方指定的是userId(站内的唯一id)，但是他想要发送的是短信。**消息推送平台内部会把userId转成手机号(这里可能是查用户团队的接口)**，但我们现在没有用户团队的接口嘛。消息推送平台内部有这种转换的话，**会方便业务方**很多。

---

### 2.9、全链路追踪
**Q：大佬好，我尝试了使用这个系统，发现全链路日志查询的时候，似乎是只能通过接收者的接收号码(邮箱号、手机号等)来查，我一开始以为是可以通过我的id来查我当天的消息发送情况，但好像没找到有这个接口。我感觉关注全链路日志的角色应该是发送者，而不是消费者？在这个全链路的查询上，我还是搞不太懂，希望大佬能讲一讲。**
**A：**我感觉关注全链路日志的角色应该是发送者，而不是消费者？为什么你会这样觉得呢？在消息推送平台来说，其实没有所谓「发送者」的概念，「使用者」的角色是被弱化的，消息推送平台关注的是「模板」，而不是具体某一次消息发送的人。
举个例子；比如我是用户团队的，现在用户要登录了，我要调用消息推送平台下发短信验证码。那这时候，我（发送者）这个是怎么标识？（难道调用下发消息接口，还得告诉接口"我"这个用户到底是谁吗？），只要在消息推送平台有对应的「模板」就可以啦。
（有了模板之后，自然就能知道该模板是哪个团队在使用的，创建该模板的人是谁 --是不用到某个具体的人身上）
全链路追踪有**两个维度**：模板和接收者

- 模板是为了统计该整体下发的情况
- 接收者是为了方便排查下发的问题（可能被哪个环境给过滤导致消息下发不到用户上）

---

### 2.10、消息模板状态
**Q：message_template的msg_status这里的40、50、60、70有点疑问  为啥要设置这几个状态  我理解这几个状态应该在sms_record体现**
**A：**消息推送平台不全是短信，其实这的状态跟 **运营的消息** 挂钩。 （运营想要知道现在模板被触发了没，调用完了没）。一般运营要监控下发的效果数据，**每发送一次就会建一个模板**，这样才好追踪数据。对于技术的模板而言，就只有**10,20,30**
![161e0221d1ae3288212a01d8c47c413.png](https://cdn.nlark.com/yuque/0/2023/png/1285871/1684476807281-6c0e5a37-6faf-4361-ba61-819d53187960.png#averageHue=%23352e2b&clientId=u2fb173c8-ff80-4&from=paste&height=363&id=ue63017c6&originHeight=363&originWidth=1548&originalType=binary&ratio=1&rotation=0&showTitle=false&size=75923&status=done&style=none&taskId=u7ff9d696-6dc6-407d-ae8b-d87ed8f7002&title=&width=1548)

---


