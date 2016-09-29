#spark 监控记录

#性能打分
##类别
* 对job进行打分,job根据stage的打分进行一次汇总
* 针对内存,并发度,数据本地性,网络
* 
###维度
* 数据大小
* 内存大小,原始数据内存的大小,原数据内存大小,使用率
* cpu个数,使用率
* 使用时长
* gc时长,task的平均gc、最大最小gc
##spark app的监控
* 数据本地的比例


##spark 集群的监控
###分类
* 针对主体分类
    > * 基于提交用户维度的监控
    > * 基于提交应用维度的监控
    
* 根据提交方式分类
    > * 定时任务
    > * 临时任务
* 操作类型分类
    > * count,save操作
    > * groupByKey,aggregateByKey
* 根据hive表分类
    > * 
* 根据读取目录来分类

###维度
* 数据大小
* 内存大小,原始数据内存的大小,原数据内存大小,使用率
* cpu个数,使用率
* 使用时长
* gc时长,task的平均gc、最大最小gc
* 字段使用排行,表使用排行
* 读取目录的次数

###收集
* spark app日志收集
* spark Listener
* hdfs 日志收集


###用途
* 宽表字段的选择(通过基于增量全体的字段的使用排行20来选择)
* 限制用户的不合理提交(平台资源是有限的,基于用户的提交数和cpu,内存使用数)
