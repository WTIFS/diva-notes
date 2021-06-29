**1. 原子性（Atomicity）**

- 原子性是指一个事务中的操作，要么全部成功，要么全部失败，如果失败，就回滚到事务开始前的状态。
- 通过什么机制来实现原子性？
  - UNDO log，事务失败时执行回滚



**2. 一致性（Consistency）**

- 一致性是指事务必须使数据库从一个一致性状态变换到另一个一致性状态，也就是说一个事务执行之前和执行之后都必须处于一致性状态。拿转账举例子，A账户和B账户之间相互转账，无论如何操作，A、B账户的总金额都必须是不变的。
- 通过什么机制实现一致性？
  - 其余的AID三个属性
  - 业务实现逻辑。如A和B的金额存在两个库，那光凭数据库根本保证不了一致性。



**3. 隔离性（Isolation）**

- 隔离性是当多个用户 并发的 访问数据库时，如果操作同一张表，数据库则为每一个用户都开启一个事务，且事务之间互不干扰，也就是说事务之间的并发是隔离的。再举个栗子，现有两个并发的事务T1和T2，T1要么在T2开始前执行，要么在T2结束后执行，如果T1先执行，那T2就在T1结束后在执行。



**4. 持久性（Durability）**

- 持久性就是指如果事务一旦被提交，数据库中数据的改变就是永久性的，即使断电或者宕机的情况下，也不会丢失提交的事务操作。
- 通过什么机制实现持久性？
  - 数据持久化
  - REDO log