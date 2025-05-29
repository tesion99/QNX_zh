# Security for Scheduler Partitions(调度分区安全)

一般默认系统上的任何人都可以添加分区并修改它们的属性。建议你使在**SchedCtl()**接口设置**SCHED_APS_ADD_SECURITY**指令或者使用**aps**命令来指定适合你系统的安全等级

如下列表展示了主要的安全选项(包括用于**aps**命令的-s选项的安全策略与对应的**SchedCtl()**标志)，按照安全升序排列。更多关于**PROCMGR_AID_APS_ROOT**生效的信息，请在C Library Reference 查看**procmgr_ability()**项

| aps         | SchedCtl()                | Description                                                  |
| ----------- | ------------------------- | ------------------------------------------------------------ |
| none        | SCHED_APS_SEC_OFF         | 系统上任何人均可添加分区并修改其属性                         |
| basic       | SCHED_APS_SEC_BASIC       | 仅仅只有运行于System分区并开启了**PROCMGR_AID_APS_ROOT**功能的进程可以改变整体调度参数。开启了**PROCMGR_AID_APS_ROOT**并运行于任何分区的进程可以设置关键预算 |
| flexible    | SCHED_APS_SEC_FLEXIBLE    | 仅仅只有运行于System分区并开启了**PROCMGR_AID_APS_ROOT**功能的进程可以改变调度参数。然后，开启**PROCMGR_AID_APS_ROOT**功能并运行于任何分区的进程可以创建子分区，将线程加入到它们所拥有的子分区，修改子分区，并修改关键预算。这使得应用可以在它们拥有的预算之外创建它们自己所有的本地子分区。预算的百分比不能为0 |
| recommended | SCHED_APS_SEC_RECOMMENDED | 仅仅只有运行于System分区并开启了**PROCMGR_AID_APS_ROOT**功能的进程可以创建分区或者修改参数。这将创建一个两级分区层次结构：System分区和它的孩子分区。仅有开启了**PROCMGR_AID_APS_ROOT**并运行于System分区的进程可以将他们自己拥有的进程加入到分区。预算的百分比不能为0 |

Note:

​	除非您正在测试分区方面并且想要在不重新启动的情况下更改所有参数，否则您应该至少设置基本的安全性。

在设置了调度分区之后，你可以使用**SCHED_APS_SEC_PARTITIONS_LOCKED**来禁止未来未认证的修改。示例如下：

```cpp
sched_aps_security_parms p;

APS_INIT_DATA( &p );
p.sec_flags = SCHED_APS_SEC_PARTITIONS_LOCKED;
SchedCtl( SCHED_APS_ADD_SECURITY, &p, sizeof(p));
```



Note:

​	在你调用**SchedCtl()**之前，请确保你初始化了相关命令数据结构的所有成员。你可以使用**APS_INIT_DATA()**宏来完成初始化操作。

以上列出的安全选项是独立选项的结合(但是使用复合选项更方便)：

```cpp
#define SCHED_APS_SEC_BASIC       (SCHED_APS_SEC_ROOT0_OVERALL | SCHED_APS_SEC_ROOT_MAKES_CRITICAL)

#define SCHED_APS_SEC_FLEXIBLE    (SCHED_APS_SEC_BASIC | SCHED_APS_SEC_NONZERO_BUDGETS |\
                                   SCHED_APS_SEC_ROOT_MAKES_PARTITIONS |\
                                   SCHED_APS_SEC_PARENT_JOINS | SCHED_APS_SEC_PARENT_MODIFIES )

#define SCHED_APS_SEC_RECOMMENDED (SCHED_APS_SEC_FLEXIBLE | SCHED_APS_SEC_SYS_MAKES_PARTITIONS |\
                                   SCHED_APS_SEC_SYS_JOINS | SCHED_APS_SEC_JOIN_SELF_ONLY)

#define SCHED_APS_SEC_OFF         0x00000000
```

这些独立选项如下：

| aps                   | SchedCtl()                          | Description                                                  |
| --------------------- | ----------------------------------- | ------------------------------------------------------------ |
| root0_overall         | SCHED_APS_SEC_ROOT0_OVERALL         | 为了修改整体调度参数(例如平均窗口大小)，你开启设置**PROCMGR_AID_APS_ROOT**并处于System分区 |
| root_makes_partitions | SCHED_APS_SEC_ROOT_MAKES_PARTITIONS | 为了创建或修改分区，你必须开启**PROCMGR_AID_APS_ROOT**       |
| sys_makes_partitions  | SCHED_APS_SEC_SYS_MAKES_PARTITIONS  | 为了创建或修改分区，你必须运行于System分区                   |
| parent_modifies       | SCHED_APS_SEC_PARENT_MODIFIES       | 允许分区被修改(SCHED_APS_MODIFY_PARTITION)，但你必须运行于被修改分区的父分区上。修改意味着改变分区百分比或者关键预算。 |
| nonzero_budgets       | SCHED_APS_SEC_NONZERO_BUDGETS       | 一个预算为0的分区不能被创建或被修改。除非你知道你的分区只需要再响应客户端请求时运行，例如接受消息，否则你应该设置该选项。 |
| root_makes_critical   | SCHED_APS_SEC_ROOT_MAKES_CRITICAL   | 为了创建一个非零关键预算或者改变一个现存的关键预算，你必须开启**PROCMGR_AID_APS_ROOT** |
| sys_makes_critical    | SCHED_APS_SEC_SYS_MAKES_CRITICAL    | 为了创建一个非零关键预算或者改变一个现存的关键预算，你必须运行于System分区 |
| root_joins            | SCHED_APS_SEC_ROOT_JOINS            | 为了将一个线程加入到一个分区，你必须开启**PROCMGR_AID_APS_ROOT** |
| sys_joins             | SCHED_APS_SEC_SYS_JOINS             | 为了加入一个线程，你必须运行于System分区                     |
| parent_joins          | SCHED_APS_SEC_PARENT_JOINS          | 你必须运行于你希望加入的分区的父分区                         |
| join_self_only        | SCHED_APS_SEC_JOIN_SELF_ONLY        | 进程只能将自身加入到分区                                     |
| partitions_locked     | SCHED_APS_SEC_PARTITIONS_LOCKED     | 禁止任何分区预算或者整体调度参数(例如窗口大小)的后续修改，设置分区后设置此项。 |



## 安全与关键线程 ##

一个线程可以设置其优先级为分区关键优先级，但是这不是一个安全问题。这是因为标记为关键的线程对线程调度器没有影响，除非线程在一个拥有关键预算的分区中。线程调度器使用安全选项来控制谁可以设置或修改一个分区的关键预算。

为了确保系统安全，防止可能的关键线程滥用。如下内容很重要：

- 仅给需要关键预算的分区赋予预算
- 尽可能多的将应用移出System分区(该分区有无穷的关键预算)