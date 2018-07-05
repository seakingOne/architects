所谓“分布式ID生成方案”，是指在分布式环境下，生成全局唯一ID的方法。
<br/>
可以利用DB自增键(auto inc id)来生成全局唯一ID，插入一条记录，生成一个ID：<br/>
![image](https://github.com/seakingOne/architects/blob/master/resource/auto_id/1.jpg)
<br/>
这个方案利用了数据库的单点特性，其优点为：<br/>
	无需写额外代码
	全局唯一
	绝对递增
	递增ID的步长确定
<br/>
其不足为：<br/>
	需要做数据库HA，保证生成ID的高可用
	数据库中记录数较多
	生成ID的性能，取决于数据库插入性能

<br/>

优化方案为：<br/>
	利用双主保证高可用
	定期删除数据
	增加一层服务，采用批量生成的方式降低数据库的写压力，提升整体性能

<br/>
增加服务后，DB中只需保存当前最大的ID即可，在服务启动初始化的过程中，首先拉取当前的max-id：![image](https://github.com/seakingOne/architects/blob/master/resource/auto_id/2.jpg)
<br/>
然后批量获取一批ID，放到id-servcie内存里，并将max-id写回数据库![image](https://github.com/seakingOne/architects/blob/master/resource/auto_id/3.jpg)
<br/>
这个方案的优点：<br/>
数据库只保存一条记录
性能极大增强
<br/>
其不足为：<br/>
如果id-service重启，可能内存会有一段已经申请的ID没有分配出去，导致ID空洞，当然，这不是一个严重的问题
服务没有做HA，无法保证高可用
<br/>
优化方案为：<br/>
冗余服务，做集群保证高可用
<br/>

冗余了服务后，多个服务在启动过程中，进行ID批量申请时，可能由于并发导致数据不一致：<br/>
![image](https://github.com/seakingOne/architects/blob/master/resource/auto_id/4.jpg)
<br/>
如上图所示，两个id-service在启动的过程中，同时拿到了max-id为100。<br/>

两个id-service同时对数据库的max-id进行写回：<br/>
![image](https://github.com/seakingOne/architects/blob/master/resource/auto_id/5.jpg)<br/>
写回max-id成功后，这两个id-service都以为自己拿到了[100,200]这一批ID，导致集群会生成重复的ID。<br/>

只要实施CAS乐观锁，在写回时对max-id的初始条件进行比对，就能避免数据的不一致，写回SQL由：<br/>
update T set max_id=200;<br/>
升级为：<br/>
update T set max_id=200 where max_id=100;<br/>

这样，id-service2写回时，就会失败：<br/>
![image](https://github.com/seakingOne/architects/blob/master/resource/auto_id/6.jpg)<br/>
失败后，id-service2要再次查询max-id：<br/>
![image](https://github.com/seakingOne/architects/blob/master/resource/auto_id/7.jpg)<br/>
此时max-id已经变为200，于是id-service2获取到了[200, 300]这一批ID，并将max-id=300写回<br/>
![image](https://github.com/seakingOne/architects/blob/master/resource/auto_id/8.jpg)<br/>


