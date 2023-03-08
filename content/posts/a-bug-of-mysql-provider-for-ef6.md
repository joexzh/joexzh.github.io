---
title:      "MySql Provider for EF6 的 bug"
date:       2017-04-23T19:00:00+08:00
draft:      false
tags:
    - Entity Framework
    - se
---

最近一个公司的项目用 MySQL 作为数据库, ORM 使用 Entity Framework 和 Dapper. 我在领域服务中选择了 EF 作为访问层 ORM, 是因为好维护和开发速度快.

    领域对应的数据表关系如下:

[![Tables](diagram_1.svg)](diagram_1.svg)

一开始导入一些简单的测试数据, 完美运行. 然而交付给系统测试之后, 噩梦开始了. 有一些 Order 操作报错了. 赶紧从测试那导一份数据回来测试. OMG, 生成的 SQL 语句竟然无法运行. 错误如下:

> Unknown column 'Extent6.Id' in 'on clause'

对应的是这一段 c# code:

```c#
var query = Context.Set<Order>().Where(o => o.Id == orderId)
   .Include(o => o.Tourists)
   .Include(o => o.OrderItems.Select(i => i.RequirementItem))
   .Include(o => o.Refunds)
   .Include(o => o.Refunds.Select(r => r.Tourists))
   .Include(o => o.Refunds.Select(r => r.RefundItems.Select(i => i.OrderItem)));
var order = await query.FirstAsync();
```

生成的 sql statement 超级长: [sql](mysql.txt)

把这段 sql 放进 Navicat 一运行, bang! 果然报了同样的错误. 这可棘手了, 生成语句不是我能控制的范围内. 是什么原因呢? google 一下关键字 "ef mysql Unknown column id", 发现早就有人在 bugs.mysql.com 提过很多类似的 issue了, 比如这个 [MySQL Bugs: #72004: Generated SQL requests column that doesnt exist](https://bugs.mysql.com/bug.php?id=72004).

mysql 开发者最后回复道:

    Fixed as of the upcoming MySQL Connector/Net 6.7.6/6.8.4/6.9.5 releases, and here's the changelog entry:

    The query optimization routine would return statements with invalid table aliases when nested queries were being optimized. This would throw an "Unknown column" exception.

出现问题是因为嵌套关系的会生成不正确的别名, MySQL 语句比 SQL Server 要严格, 有时候外层语句定义的别名, 在内层无法识别.

然而我目前用的是 6.9.9 版本了, 甚至连名字都改成了 MySql.Data.Entity.EF6, 怎么还有这个bug? 既然上面提到 6.9.5 版本会修复, 那我就换成 6.9.5 版本呗. 然而, 在 nuget 上面无论 MySql.ConnectorNET.Entity 还是 MySql.Data.Entity.EF6, 都没有发布上面说会修复的版本 :joy:.

没有办法了, 只能退而求其次, 把 c# 代码改为手动 lazy load 的方式, 牺牲了性能, 只能用 "更改领域状态必须严谨, 速度慢点也没关系" 自我安慰了:

```c#
var query = Context.Set<Order>().Where(o => o.Id == orderId)
   .Include(o => o.Tourists)
   .Include(o => o.OrderItems.Select(i => i.RequirementItem))
   //.Include(o => o.Refunds);
   //.Include(o => o.Refunds.Select(r => r.Tourists))
   //.Include(o => o.Refunds.Select(r => r.RefundItems.Select(i => i.OrderItem)));
// ↑ above three lines will generate improper sql in mysql :(. So it's the only choice to using lazy load manually

 var order = await query.FirstAsync();

 // manually using lazy load
 Context.Entry(order).Collection(o => o.Refunds).Query()
     .Include(r => r.Tourists)
     .Include(r => r.RefundItems.Select(i => i.OrderItem))
     .Load();
```

单元测试通过后, 又交付给系统测试了. 但是后面几天简直寝室难安, 这无疑是个定时炸弹, 以后项目当然还要用 MySQL + EF, 万一有人忘了这一点, 随时又可能牵起一阵腥风血雨. 在 MySql.Data.Entity.EF6 没改善这个问题之前, 不能坐以待毙. 最近 .Net Core 出稳定版了, 那么 Entity Framework Core + MySql.Data.EntityFrameworkCore 表现如何. 马上试一下.

## 把项目换成 .Net Core

不是马上把线上的项目换掉, 先开个分支测试, 毕竟现在 MySql.Data.EntityFrameworkCore 还处于 pre-release 阶段: [MySQL: Microsoft Docs](https://docs.microsoft.com/en-us/ef/core/providers/mysql/).

经过一番折腾之后...

Ooh, 我放弃了, 目前 EF Core 还没支持 ComplexType, 项目中很多地方都用它来做 ValueObject. 但是可以用另一种 Mapping 方式来实现, 看下文.

关于 EF Core ComplexType 的 issue: [Complex types and/or value objects · Issue #246 · aspnet/EntityFramework](https://github.com/aspnet/EntityFramework/issues/246#issuecomment-241813753).

这是一位 EF Core 的主要贡献者针对 ComplexType 起草的长篇大论:

* 提到了不同人对 EF 的 ComplextType 或 DDD 的 ValueObjects 理解也会不同.
* ComplexType 没有主键, 它和其它属性没什么区别, 诸如你可以把整个 ComplexType 序列化成 JSON, 然后存到一列里面.
* 在 Mapping 方面目前 EF6 和 EF Core 可以分别用一下的做法:

```c#
// EF6
modelBuilder.ComplexType<CourseDetails>() 
    .Property(t => t.Location) 
    .HasMaxLength(20);
```

```c#
// EF Core
modelBuilder.Entity<OnsiteCourse>() 
    .Property(t => t.Details.Location) 
    .HasMaxLength(20);
```

* 以后的 EF Core 的 ComplexType 可能实现很多特性
  * Nullability: 可配置读取出来后设置为空
  * Navigation properties: 可包含 FK 和 Navigation properties
  * Query: ComplexType 中的属性可查询 (这就是我使用 ComplexType 的原因, ValueObject 和 可查询2者兼备)
  * Nesting: 可嵌套
  * Keys: 可当做 PK 的一部分
  * Sharing instances
  * Mutability and change tracking (大部分人都是设置成 Immutable 吧?)

然后, 他还@了其他 EF Core Team 的主要成员, 要大家讨论之前先看看他写的东西, 蛤蛤, 应该是一位 leader 吧.

## 结局

扯远了, 看了很多 EF Core 的东东. 就当顺路学习吧 :).

对这次问题, 在没有新方案之前, 应该

* 三层以上的关系, 使用 sql join 代替 include
* 或者手动 lazy load (尽量少少少用)

最后, 测试项目地址: [EFMySqlBugTest](https://github.com/joexzh/EFMySqlBugTest), 下次会尝试用新的 Mapping 方式测试.

    To be continued...
