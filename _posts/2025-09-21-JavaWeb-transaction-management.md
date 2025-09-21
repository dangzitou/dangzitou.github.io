---
title: "JavaWeb学习记录-Spring事务管理"
date: 2025-09-21 18:00:00 +0800
categories: [Java, 学习记录]
tags: [Spring, 事务管理, JavaWeb, 数据库]
---
## 前言
今天是JavaWeb学习的第十天，进行首个小型项目——tlias教学管理系统开发的第五天，正在开发员工的新增功能。
这个时候就要用到Spring框架中的事务管理功能了，因为比如说下面的代码：
```java
emp.setCreateTime(LocalDateTime.now());
emp.setUpdateTime(LocalDateTime.now());
empMapper.insert(emp);
int i = 1/0;    //模拟异常

List<EmpExpr> exprList = emp.getExprList();
if(!CollectionUtils.isEmpty(exprList)){
    for(EmpExpr expr : exprList){
        expr.setEmpId(emp.getId());
    }
    empExprMapper.insertBatch(exprList);
}
```
在新增员工数据的时候，如果出现如上模拟异常，就会出现员工表被插入了数据，但是员工经验表没有插入数据的情况，破坏了数据的一致性。所以如果像银行转账之类的操作，如果扣了款，但因为某些异常没到另一个人账上，就会出现严重的问题。
所以Spring 事务管理是企业级 Java 应用开发中非常重要的一环，能够保证数据操作的一致性和完整性。本文将简要记录我学习 Spring 事务管理的主要内容和实践经验。

## 1. 什么是事务

事务（Transaction）是指一组操作，要么全部成功，要么全部失败。事务具有四大特性（ACID）：

- **原子性（Atomicity）**：事务中的所有操作要么全部完成，要么全部不做。
- **一致性（Consistency）**：事务完成后，数据从一个一致性状态转变为另一个一致性状态。
- **隔离性（Isolation）**：多个事务并发执行时，彼此之间不会互相影响。
- **持久性（Durability）**：事务一旦提交，对数据的修改就是永久性的。

## 2. Spring 事务管理方式

Spring 提供了两种主要的事务管理方式：

- **编程式事务管理**：通过代码手动管理事务，灵活但侵入性强。
- **声明式事务管理**：通过配置或注解方式实现，推荐使用，解耦业务逻辑和事务管理。

常用注解：`@Transactional`

## 3. 事务传播行为

Spring 支持多种事务传播行为，常见的有：

- `REQUIRED`：默认值，当前存在事务则加入，否则新建事务。
- `REQUIRES_NEW`：每次都新建一个事务，原有事务挂起。
- `SUPPORTS`：如果有事务则加入，没有则以非事务方式执行。

这是用的比较多的三种

## 4. 事务隔离级别

常见隔离级别：

- `READ_UNCOMMITTED`
- `READ_COMMITTED`
- `REPEATABLE_READ`
- `SERIALIZABLE`

隔离级别越高，安全性越高，但并发性能越低。

## 5. 在tlias项目中的应用

```java
//Transaction一般加在多次数据库操作的方法上，维持数据的一致性
//事务管理：只有运行时异常才回滚（RuntimeException及其子类），所以要加上rollbackFor
@Transactional(rollbackFor = {Exception.class}) 
@Override
public void save(Emp emp) {
    try {
        emp.setCreateTime(LocalDateTime.now());
        emp.setUpdateTime(LocalDateTime.now());
        empMapper.insert(emp);
        //int i = 1/0;//模拟异常

        List<EmpExpr> exprList = emp.getExprList();
        if(!CollectionUtils.isEmpty(exprList)){
            for(EmpExpr expr : exprList){
                expr.setEmpId(emp.getId());
            }
            empExprMapper.insertBatch(exprList);
        }
    } finally {
        //记录操作日志
        //避免因为日志记录失败而导致主事务回滚,所以这里开启一个新事务
        EmpLog empLog = new EmpLog(null, LocalDateTime.now(), "新增" + emp);
        empLogService.insertLog(empLog);
    }
}
```

## 6. 常见问题

- 事务只对运行在 Spring 容器管理下的 Bean 有效
- 默认仅运行时异常（RuntimeException）回滚
- 事务失效常见原因：自调用、未被 Spring 管理等

---
