## 记录一次InnoDB update的事务执行过程

最近整理了MySQL索引，事务，MVCC，日志系统等相关知识，想着具体结合某个操作的执行过程来串串。
今天刚好碰见个update操作，就以这个操作为例，记录一下InnoDB的事务执行过程。
这个过程涉及到了多个核心组件，包括Buffer Pool，Undo Log，Redo Log，binlog，以及MVCC等。

为了直观理解，可以将整个流程分为几个核心阶段：

```mermaid
flowchart TD
    A[客户端发起 UPDATE 语句] --> B
    
    subgraph B [执行阶段]
        B1[SQL解析与优化<br>生成执行计划]
        B1 --> B2[进入 InnoDB 引擎层]
        B2 --> B3[查 Buffer Pool 缓存]
        B3 -->|命中| B4[在内存中锁定并修改数据页]
        B3 -->|未命中| B5[从磁盘加载数据页到 Buffer Pool]
        B5 --> B4
        B4 --> B6[写 Undo Log<br>（用于回滚和MVCC）]
        B4 --> B7[写 Redo Log Buffer<br>（记录物理修改）]
    end

    B --> C
    
    subgraph C ["提交阶段 (2PC)"]
        C1[刷 Redo Log Buffer 到磁盘<br>（Prepare阶段）]
        C2[写 Binlog 到磁盘]
        C3[刷 Redo Log 到磁盘<br>（Commit阶段）]
    end

    C --> D[返回客户端成功]

    C1 -.-> E[后台异步任务]
    C3 -.-> E

    subgraph E [后台处理]
        E1[将脏页刷回磁盘<br>（Checkpoint机制）]
    end
```

### 1. 事务开始

显示执行BEGIN/START TRANSACTION，开启一个事务。
InnoDB会为事务分配一个事务ID（后续就会记录在数据表的隐藏列-DB_TRX_ID 中），初始化事务上下文。事务ID是全局唯一的，单调递增的。

### 2. 定位并加载数据页

InnoDB根据where条件，通过**索引**定位目标行，加载数据页到Buffer Pool(缓冲池)。
检查Buffer Pool中是否已经存在该数据页，如果存在，则直接返回数据页；如果不存在，则从磁盘读取数据页到Buffer Pool。
 
### 3. 加行级锁

InnoDB 

