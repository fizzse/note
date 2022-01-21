# SELECT语句怎么执行

## MYSQL SEVER 结构构成
- sever层+存储引擎构成（innoDb，mySam引擎等），sever层组件如下
- 链接器：客户端建立链接，确定链接权限
- 分析器：分析语句，语法
- 优化器：优化语句，选择最优的索引，与join方式
- 执行器：执行语句，返回结果

## MYSQL日志
- redo log：引擎层日志，innoDb特有
- binlog：sever层日志。逻辑日志row模式下为sql语句
- 当有数据写入时，写redolog prepare，更新内存，写binlog，redolog commit（两阶段提交）。空闲时落盘