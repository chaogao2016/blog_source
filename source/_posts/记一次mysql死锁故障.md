---
title: 记一次mysql死锁故障
date: 2020-09-15 09:08:50
tags: [debug,数据库,死锁]
---
## 这是生产环境的一个写库调用产生的问题，在这里记下解决过程。

## 现象
&emsp;&emsp;我有一个golang写的常驻后台服务，每隔十多秒就会调用一个php实现的接口。而这个php实现的接口中有一个长事务，其中包含了对若干张表的update操作，因为这十多个操作都在一个事务中，所以他们是原子性的，要么一起成功，要么一起失败。

&emsp;&emsp;大部分时间里这个接口都工作的很好，但是有较小的几率，会出现报错：
<!--more-->
```
{
    "err":"批量为多个用户修改账户并添加多个礼物接口访问失败:1213:Deadlock found when trying to get lock; try restarting transaction
 [ SQL语句 ] : UPDATE `ln_user` SET `money`=money+1200 WHERE `user_id` = 646598",
    "level":"error",
    "line":"logic/logic.go:69",
    "msg":"存储游戏结果失败！",
    "time":"2020-09-14T20:17:31+08:00",
    "type":"db"
}
```
&emsp;&emsp;日志是golang这边打印的，但是真正发生错误的位置在php接口中，很明显的描述，发生了死锁(Deadlock)。

&emsp;&emsp;很少遇见这种问题，我需要温习一下我的数据库知识。

## 事务的定义
&emsp;&emsp;**事务，在计算机术语中是指访问并可能更新数据库中各种数据项的一个程序执行单元(unit)**。

&emsp;&emsp;单元其实就是原子性

&emsp;&emsp;事务应该具有4个属性：原子性、一致性、隔离性、持久性。这四个属性通常称为ACID特性。
 - 原子性（atomicity）。一个事务是一个不可分割的工作单位，事务中包括的操作要么都做，要么都不做。
 - 一致性（consistency）。事务必须是使数据库从一个一致性状态变到另一个一致性状态。一致性与原子性是密切相关的。
 - 隔离性（isolation）。一个事务的执行不能被其他事务干扰。即一个事务内部的操作及使用的数据对并发的其他事务是隔离的，并发执行的各个事务之间不能互相干扰。
 - 持久性（durability）。持久性也称永久性（permanence），指一个事务一旦提交，它对数据库中数据的改变就应该是永久性的。接下来的其他操作或故障不应该对其有任何影响。

## 死锁的定义
&emsp;&emsp;**死锁是指两个或两个以上的进程在执行过程中，由于竞争资源或者由于彼此通信而造成的一种阻塞的现象，若无外力作用，它们都将无法推进下去。此时称系统处于死锁状态或系统产生了死锁，这些永远在互相等待的进程称为死锁进程**。

&emsp;&emsp;**例如，如果进程A锁住了记录1并等待记录2，而进程B锁住了记录2并等待记录1，这样两个进程就发生了死锁现象**。

&emsp;&emsp;我们知道了事务和死锁的广义定义。那么，具体到mysql数据库，什么情况下会发生死锁呢。

&emsp;&emsp;按照定义，前人总结出了产生死锁的四个必要条件：
 1. 互斥条件：一个资源每次只能被一个进程使用。
 2. 请求与保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放。
 3. 不剥夺条件:进程已获得的资源，在末使用完之前，不能强行剥夺。
 4. 循环等待条件:若干进程之间形成一种头尾相接的循环等待资源关系。

&emsp;&emsp;现在我们了解了死锁是什么，以及死锁会在什么情况下发生，接下来我们要分析代码，找到触发死锁的原因。

## 接口逻辑
&emsp;&emsp;首先我们找到php接口代码，我接下来用伪代码展现接口逻辑这是一个批量修改用户货币并且给用户批量加道具的功能。
```
开启事务
for 用户道具数组 as 用户,道具列表 {
    // 这一步有数据库写操作
    用户.修改金币()

    for 道具列表 as 道具 {
        // 这一步有数据库写操作
        用户.添加道具(道具)
    }
}
提交事务
if 提交事务失败 {
    回滚事务
}
```

&emsp;&emsp;在死锁这个场景下，我们需要着重关注点是，在一个事务中的update操作，在事务提交前，会不会对要update的资源加锁，这个问题很重要，如果它会，那么死锁的四个必要条件就都有可能达成。

&emsp;&emsp;其实这个很简单的实验就能完成，事务中update某一行数据并sleep，然后看看还能不能修改数据。

&emsp;&emsp;答案是会加锁，加的是排他锁。

&emsp;&emsp;这里推荐一篇文章，对死锁的概念和性质做了很详细的介绍。
- https://zhuanlan.zhihu.com/p/36060546

&emsp;&emsp;我们上面说的就是文章中描述的隐式锁，实际还有显式锁。

&emsp;&emsp;好了，到这里我们可以得出结论，这段代码是由可能发生能死锁的，譬如有另一个地方，修改用户金币的顺序不一样，就会导致两个事务互相等待资源，触发死锁。

## 分析日志
&emsp;&emsp;接下来我们要找到到底是哪个sql哪里触发了这个死锁。

&emsp;&emsp;在mysql上执行show engine innodb status，会得到最近的死锁日志。

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2020-09-14 20:17:31 0x7f8bb38f4700
*** (1) TRANSACTION:
TRANSACTION 494692805, ACTIVE 0 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 9 lock struct(s), heap size 1136, 6 row lock(s), undo log entries 4
MySQL thread id 8351333, OS thread handle 140237990774528, query id 50021361 172.16.92.59 yutangyuyin updating
UPDATE `ln_user` SET `money`=money+1200 WHERE `user_id` = 646598
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 687 page no 1512 n bits 136 index PRIMARY of table `yutangyuyin`.`ln_user` trx id 494692805 lock_mode X locks rec but not gap waiting
Record lock, heap no 30 PHYSICAL RECORD: n_fields 35; compact format; info bits 0
 0: len 4; hex 0009ddc6; asc     ;;
 1: len 6; hex 00001d7c69be; asc    |i ;;
 2: len 7; hex 49000086950e7b; asc I     {;;
 3: len 11; hex 3133323331363330373137; asc 13231630717;;
 4: len 0; hex ; asc ;;
 5: len 6; hex 333437383733; asc 347873;;
 6: len 4; hex 80000000; asc     ;;
 7: len 1; hex 01; asc  ;;
 8: len 4; hex ded9f424; asc    $;;
 9: len 4; hex ded9f424; asc    $;;
 10: len 3; hex 800001; asc    ;;
 11: len 4; hex 8000001e; asc     ;;
 12: SQL NULL;
 13: len 0; hex ; asc ;;
 14: len 1; hex 80; asc  ;;
 15: len 30; hex 316366623565363064353635366630313338343462326531363336623161; asc 1cfb5e60d5656f013844b2e1636b1a; (total 32 bytes);
 16: len 10; hex 55304766754d47633277; asc U0GfuMGc2w;;
 17: len 9; hex 800000000000031300; asc          ;;
 18: len 9; hex 800000000000000000; asc          ;;
 19: len 4; hex 80000001; asc     ;;
 20: len 4; hex 80000000; asc     ;;
 21: len 9; hex 800000000000000132; asc         2;;
 22: len 4; hex 80000000; asc     ;;
 23: len 0; hex ; asc ;;
 24: len 0; hex ; asc ;;
 25: len 4; hex df5f5526; asc  _U&;;
 26: len 4; hex 80000000; asc     ;;
 27: len 1; hex 81; asc  ;;
 28: len 9; hex 800000000000000000; asc          ;;
 29: len 9; hex 800000000000000000; asc          ;;
 30: len 7; hex 64656661756c74; asc default;;
 31: len 1; hex 80; asc  ;;
 32: len 4; hex 80000000; asc     ;;
 33: len 4; hex 80000000; asc     ;;
 34: len 8; hex 8000000000000000; asc         ;;

*** (2) TRANSACTION:
TRANSACTION 494692798, ACTIVE 0 sec fetching rows
mysql tables in use 3, locked 3
23 lock struct(s), heap size 3520, 13 row lock(s), undo log entries 18
MySQL thread id 8351329, OS thread handle 140237989693184, query id 50021359 172.16.92.58 yutangyuyin updating
UPDATE `ln_gift_backpack` SET `number`=number-9 WHERE `user_id` = 811036 AND `gift_id` = 310 AND `number` >= 9
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 687 page no 1512 n bits 136 index PRIMARY of table `yutangyuyin`.`ln_user` trx id 494692798 lock_mode X locks rec but not gap
Record lock, heap no 30 PHYSICAL RECORD: n_fields 35; compact format; info bits 0
 0: len 4; hex 0009ddc6; asc     ;;
 1: len 6; hex 00001d7c69be; asc    |i ;;
 2: len 7; hex 49000086950e7b; asc I     {;;
 3: len 11; hex 3133323331363330373137; asc 13231630717;;
 4: len 0; hex ; asc ;;
 5: len 6; hex 333437383733; asc 347873;;
 6: len 4; hex 80000000; asc     ;;
 7: len 1; hex 01; asc  ;;
 8: len 4; hex ded9f424; asc    $;;
 9: len 4; hex ded9f424; asc    $;;
 10: len 3; hex 800001; asc    ;;
 11: len 4; hex 8000001e; asc     ;;
 12: SQL NULL;
 13: len 0; hex ; asc ;;
 14: len 1; hex 80; asc  ;;
 15: len 30; hex 316366623565363064353635366630313338343462326531363336623161; asc 1cfb5e60d5656f013844b2e1636b1a; (total 32 bytes);
 16: len 10; hex 55304766754d47633277; asc U0GfuMGc2w;;
 17: len 9; hex 800000000000031300; asc          ;;
 18: len 9; hex 800000000000000000; asc          ;;
 19: len 4; hex 80000001; asc     ;;
 20: len 4; hex 80000000; asc     ;;
 21: len 9; hex 800000000000000132; asc         2;;
 22: len 4; hex 80000000; asc     ;;
 23: len 0; hex ; asc ;;
 24: len 0; hex ; asc ;;
 25: len 4; hex df5f5526; asc  _U&;;
 26: len 4; hex 80000000; asc     ;;
 27: len 1; hex 81; asc  ;;
 28: len 9; hex 800000000000000000; asc          ;;
 29: len 9; hex 800000000000000000; asc          ;;
 30: len 7; hex 64656661756c74; asc default;;
 31: len 1; hex 80; asc  ;;
 32: len 4; hex 80000000; asc     ;;
 33: len 4; hex 80000000; asc     ;;
 34: len 8; hex 8000000000000000; asc         ;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 318 page no 140 n bits 512 index PRIMARY of table `yutangyuyin`.`ln_gift_backpack` trx id 494692798 lock_mode X locks rec but not gap waiting
Record lock, heap no 37 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 8014df5d; asc    ];;
 1: len 6; hex 00001d7c69c5; asc    |i ;;
 2: len 7; hex 4f0000868e15d4; asc O      ;;
 3: len 4; hex 800907bf; asc     ;;
 4: len 4; hex 80000136; asc    6;;
 5: len 4; hex 8000059d; asc     ;;
 6: len 1; hex 81; asc  ;;

*** WE ROLL BACK TRANSACTION (1)
------------
TRANSACTIONS
------------
```

&emsp;&emsp;这个日志的信息其实很有限，我们可以看到，触发死锁时事务1首先持有用户表的锁，然后事务2持有背包表的锁，各有一个等待锁释放的过程，然后死锁，回滚了。有用的信息比较少，但我们还是找到了一些证据。

&emsp;&emsp;事务2的sql并不是我在接口里的sql，我需要找到这个sql调用的地方，看看这个事务是不是也锁了user表，基本就可以确定了。

&emsp;&emsp;这里推荐一篇文章，对分析日志做了很详细的注解。
 - https://www.aneasystone.com/archives/2018/04/solving-dead-locks-four.html

## 解决方案
&emsp;&emsp;有一些通用做法可以让我们尽可能避开死锁。

- 以固定的顺序访问表和行。比如两个更新数据的事务，事务A 更新数据的顺序 为1，2；事务B更新数据的顺序为2，1。这样更可能会造成死锁。
- 大事务拆小。大事务更倾向于死锁，如果业务允许，将大事务拆小。
- 在同一个事务中，尽可能做到一次锁定所需要的所有资源，减少死锁概率。
- 降低隔离级别。如果业务允许，将隔离级别调低也是较好的选择，比如将隔离级别从RR调整为RC，可以避免掉很多因为gap锁造成的死锁。
- 为表添加合理的索引。可以看到如果不走索引将会为表的每一行记录添加上锁，死锁的概率大大增大。

&emsp;&emsp;不过我面临的是遗留代码不好修改和工期的双重压力，所以我选择了更简单的方法，重试。

&emsp;&emsp;mysql的官方手册是这样解释的:
```
// 如果事务因死锁而失败，请始终做好重新发出事务的准备。死锁并不危险。再试一次。 
Always be prepared to re-issue a transaction if it fails due to deadlock. Deadlocks are not dangerous. Just try again.
```
&emsp;&emsp;部署后暂时没有发现问题，再观察一段时间。

## 总结
&emsp;&emsp;典型的并发问题，没有一定的用户数量的系统，很少会遇到这样的问题。

&emsp;&emsp;这也是很好的学习机会，挑战才能让人进步，不能被困难的问题吓到。

&emsp;&emsp;发现问题，描述现象，查找资料，理解原理，复现问题，解决问题。
