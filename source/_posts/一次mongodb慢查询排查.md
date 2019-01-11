---
title: 一次mongodb慢查询排查
date: 2019-01-10 23:59:05
tags: [技术,数据库]
categories: 技术
---

## 背景

前几天晚上下班后，同事让我帮忙把mongodb里**一个月前**的用户通知信息挪走，因为mongodb一直在报警，他觉得是信息数量太多，慢查询导致的。因为当时已经下班，他电脑也没电了，所以打算临时解决。

**我觉得还没到迫不得已，非得操作数据的时候，所以就有了这次慢查询排查**

## 定位问题

因为业务不是我写的，所以先是找到代码，大致了解查询语句的功能，下面是代码节选:

```javascript
const filter = [
    {
      $match: {
        who,
        sendTo: { $in: tags },
        exceptSendTo: { $nin: tags },
        msgTime: {
          $lt: Math.ceil(Date.now() / 1000),
        },
        isInvalid: { $exists: false },
      },
    },
    { $sort: { msgTime: -1 } },
    {
      $group: {
        _id: groupBy,
        doc: { $first: '$$ROOT' },
      },
    },
  ];
log.info('filter:', JSON.stringify(filter));
const rows = yield notificationCol.aggregate(filter);
```

这段代码主要是为了查询用户聊天列表的最新消息，这里不纠结具体业务实现方式是否正确。

## 第一轮

看到慢查询，一开始想到的是索引问题，查了下，who和sendTo已经加了索引，msgTime也有索引，且分别单独查都很快，而这句聚合查询需要至少一秒。

所以我把注意力集中在了group上，确认了业务上不能去掉group操作之后，一下好像就陷入了困境。

然后我想起同事让我挪走一个月前的消息，既然业务层允许这么操作，那在这里我同样也可以这么做呀。于是在match里，我加了msgTime一个月的判断，试了下耗时降到了300ms。

虽然算不上彻底解决，但也算解了燃眉之急，就把代码提交去洗澡了。

## 第二轮

> 洗澡的时候越想越奇怪，为什么在聚合中group索引会失效呢？

洗完澡，我就试着把group去掉，发现速度并没有提升，这下子就有意思了。我又分别试了match+group，sort+group的组合，速度都很快。那很明显就是match和sort组合的原因了。

但是不管match和sort哪个在前面，速度都很慢，再次陷入困境。我开始翻资料，查各种mongo聚合里索引的优化，但都没有查到有用的说法。但是却发现并确信，**mongodb聚合里的sort是可以利用索引的**。

我用explain查语句的查询情况，发现确实是有利用索引的，使用的是msgTime的索引，而没有用who+sendTo的索引。翻看手册，mongodb聚合是可以指定索引的，但是当前使用的mongo版本却不支持。

关于索引的，想起前阵子看的mysql的索引，猜测也许mongo也可以人为诱导索引的选择，于是我在sort里又加了who和sendTo的排序条件，瞬间打开了新世界的大门，查询耗时成功降了下来，mongodb选择了who和sendTo作为索引。

## 结论

**还是要逼着自己多思考，每一次将就都是在逃避挑战。**

**思路决定行为，下次遇到慢查询，应该先用explain**



