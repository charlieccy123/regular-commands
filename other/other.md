# 其他


## 发布策略

滚动发布：直接对现有版本实例进行更新，每次更新比例可以控制，比如启动一部分新版本然后下掉一部分老版本，逐步完成所有版本替换，这个过程最大的问题不是比例问题，而是生产流量问题当更新开始以后新版本一旦启动运行那么就开始接入了生产流量，这时候一旦发现问题你很难判断是新版本导致的还是老版本导致的，虽然大概率是新版本可也并不能排除老版本的嫌疑。这时候其实已经影响了线上稳定性和可用性，回滚其实干掉新版本启动老版本实例到既定个数。注意，传统蓝绿发布是新版本启动相同个数实例然后直接切流100%到新版本，有问题再切流回来。

蓝绿发布：老版本数量不变，启动同等数量的新版本，然后逐步小比例切流量到新版本，或者通过流量标签的方式让测试进行验证，观察没有问题时，再逐步放大流量直到100%流量切到新版本，整个过程中老版本并没有下掉，如果发现问题可以直接切流量回去到老版本上实现快速回滚大大降低了回滚时间。流量都切到新版本后没有问题再下掉老版本实力。当然流量全部切到新版本后老版本要保留多久要看服务重要程度，当然并不会并存很长时间。毕竟它消耗的是双倍的资源。

灰度发布：也叫做金丝雀发布，老版本数量不变，启动一个新版本实力，这样线上大部分流量走老版本，一小部分流量走新版本，没有问题之后再全量替换。

一般都是预发环境部署，接生产数据不接生产流量，测试通过流量标签访问新版本，功能验证没有问题，走灰度发布接生产流量，有问题就及时下掉，老版本全量都在所以不受影响，回滚快速。如果没有问题逐步
扩大新版本实例个数放大流量（这时候流量比例提高，老版本实例依然都在），当新版本流量到达100%后，继续观察没有问题，则下掉老版本，如果有问题可以及时切流量到老版本。

灰度发布期间能发现的问题有限。如数据库慢查问题、死锁问题等，10%流量很难发现，可能只会在100%流量中才容易暴露，所以这就是为什么灰度发布观察日志和指标没明显异常后扩大实例个数以及提高流量比例
逐渐到100%，这样是为了暴露小流量时不容易发现的问题。

基于argocd的蓝绿和金丝雀发布，https://blog.csdn.net/weixin_57766513/article/details/129754975