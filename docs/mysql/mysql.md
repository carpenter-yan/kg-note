# mysql知识点  

## 数据库理论  

#### 事务特性  
  - 原子性（Atomicity）  
  - 一致性（Consistency）  
  - 隔离性（Isolation）  
  - 持久性（Durability）  
#### 并发一致性  
  - 丢失修改  
  - 读脏数据  
  - 不可重复读  
  - 幻影读  
#### 隔离级别  
  - 未提交读(read uncommitted)  
  - 提交读(read committed)  
  - 不可重复读(repeatable read)  
  - 可串行化(serializable)  
  - 隔离级别能解决的并发一致性问题  
#### 封锁  
- 封锁粒度  
  - 行级锁和表级锁  
  - 锁开销和并发程度  
- 封锁类型  
  - 读写锁  
    - 互斥锁（Exclusive），简写为 X 锁，又称写锁  
    - 共享锁（Shared），简写为 S 锁，又称读锁  
  - 意向锁  
    - IS表锁  
    - IX表锁  
- 封锁协议  
  - 三级封锁协议
    - 1级封锁协议  
    - 2级封锁协议  
    - 3级封锁协议  
  - 两段锁协议  
  - 隐式与显示锁  

## 索引相关
## 优化相关
#### 参考资料
- [CyC2018数据库理论](https://github.com/CyC2018/CS-Notes/blob/master/notes/%E6%95%B0%E6%8D%AE%E5%BA%93%E7%B3%BB%E7%BB%9F%E5%8E%9F%E7%90%86.md)  