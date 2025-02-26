REQUIRED
如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。
方法1 方法2 都加了@Transactional注解
方法1 调用方法2 方法2 insert后报错，方法1 和方法2 的内容都会回滚
SUPPORTS
如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
方法1 不加@Transactional注解 方法2 @Transactional 注解propagation = Propagation.SUPPORTS
方法1 调用方法2 方法2 insert后报错，方法1 的内容不会回滚

方法1 加@Transactional注解 方法2 @Transactional 注解propagation = Propagation.SUPPORTS
方法1 调用方法2 方法2 insert后报错，方法1 的内容会回滚

MANDATORY
如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常
方法1 不加@Transactional注解 方法2 @Transactional 注解propagation = Propagation.MANDATORY
运行时程序会报错

REQUIRES_NEW
创建一个新的事务,如果当前存在事务,则把当前事务挂起
方法1 加@Transactional注解 方法2 @Transactional 注解propagation = Propagation.REQUIRES_NEW
方法1 调用方法2 方法1 insert后报错，方法1 的内容会回滚 方法2的内容不会回滚

NOT_SUPPORTED
以非事务方式运行，如果当前存在事务，则把当前事务挂起。
方法1 不加@Transactional注解 方法2 @Transactional 注解propagation = Propagation.NOT_SUPPORTED
方法1 调用方法2 方法1 insert后报错，方法1 方法2 的内容都不会回滚

方法1 加@Transactional注解 方法2 @Transactional 注解propagation = Propagation.NOT_SUPPORTED
方法1 调用方法2 方法1 insert后报错，方法1 的内容会回滚 方法2的内容不会回滚

NEVER
以非事务方式运行，如果当前存在事务，则抛出异常。
方法1 不加@Transactional注解 方法2 @Transactional 注解propagation = Propagation.NEVER
方法1 调用方法2 方法1 insert后报错，方法1 方法2 的内容都不会回滚

方法1 加@Transactional注解 方法2 @Transactional 注解propagation = Propagation.NEVER
方法1 调用方法2 方法1 insert后报错，方法1 方法2 的内容都不会入库

NESTED
如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于REQUIRED

方法1 不加@Transactional注解 方法2 @Transactional 注解propagation = Propagation.NESTED
方法1 调用方法2 方法1 insert后报错，方法1 方法2 的内容都不会回滚

方法1 加@Transactional注解 方法2 @Transactional 注解propagation = Propagation.NESTED
方法1 调用方法2 方法1 insert后报错，方法1 方法2 的内容都会回滚