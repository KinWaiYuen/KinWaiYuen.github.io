## redolog
redo log称为重做日志，用来保证事务的原子性和持久性。
redo恢复提交事务修改的页操作
redo通常是物理日志，记录的是页的物理修改操作
确保事务的`持久性`
防止在发生故障的时间点,尚有脏页未写入磁盘，在重启mysql服务的时候，根据`redolog`进行重做，从而达到事务的`持久性`这一特性。
## undolog

undolog用来保证事务的一致性
undo回滚行记录到某个特定版本时候,使用的是`undolog`当时的记录.在进行事务操作前,`undolog`记录下db数据,保证事务`原子性`

## redolog和binlog区别
- `redolog`是innodb引擎写的,`binlog`是mysql框架写的
- `redolog`写的是页数据,`binlog`写的是指令