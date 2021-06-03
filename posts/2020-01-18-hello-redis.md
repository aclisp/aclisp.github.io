# 《Redis in Action》一本不容错过的书

## 第一章

第一章小结：

> 当你阅读时，注意 Redis 是如何改变你解决问题的方法的。你会发现在思考有关数据驱动难题时，思路从“我该如何适配表和行？”变成“我该采用何种数据结构？”

第一章范例：

类似 Stack Overflow 那样的发帖子社区

* 每个帖子信息用 hash 存储
* 浏览时，按时间、或者热度分页
* 用 zset 保存时间/热度帖子

给帖子分组，每组是同一个主题

* 每个主题是 set
* 按主题浏览帖子用 ZINTERSTORE

提示：

1. `zset` 存 top-scoring or most recent items
1. 如果帖子只有一两个组，直接按组创建 zset，无需引入 set
1. redis-cli 的结果为 `(empty list or set)` 时，redis.Do 结果为 empty slice, nil err

## 第二章

第二章范例：

购物网站，能用到 redis 的

* 登录态 session
* 购物车
* 浏览页缓存

提示：

1. 根据存储在 redis 里的数据统计，决定哪些浏览页能被缓存
1. `zset` 保存 top-N
1. ZINTERSTORE 把 `zset` 里所有值减半，以使得新计数上榜


