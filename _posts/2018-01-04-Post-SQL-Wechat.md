---
layout:       post
title:        "如何写 SQL 语句把 Wechat 的最近会话数据从数据库加载出来"
subtitle:     ""
date:         2018-01-04 00:00:00
author:       "Guang1234567"
header-img:   "img/home-bg-o.jpg"
header-mask:  0.3
catalog:      true
multilingual: false
music-enable: true
categories:
    -
    -
tags:
    - SQL
    - Wechat
---


> 此文章不允许转载, 违者必究...

> 目的: 如何写 SQL 语句把 Wechat 的`最近会话`数据从数据库加载出来.

**目录:**

* content
{:toc}


## 加载下面截图所需的数据

> 补充一点: 下面的`最近会话`需要按最后一条消息的时间排序.

![截图]({{ site.baseurl }}/img/in-post/2018-01-04-Post-SQL-Wechat/timg.jpg)


## 数据库建模

> 补充一点: 使用的是 sqlite3

**tb_conversation 表**

| 字段       | 含义           |
| ------------- |:-------------:|
| _id      | 主键 |
| chat_with      | 与谁聊天 |
| type      | 0 私聊, 1 群组 ...|


**tb_chat_message 表**

| 字段       | 含义           |
| ------------- |:-------------:|
| _id      | 主键 |
| msg_id      | 消息的唯一ID, 是个 UUID |
| chat_with      | 与谁聊天 |
| timestamp      | 消息的时间戳 |
| type      | 0 文本, 1 图片, 3 复合消息 ...|


**tb_chat_message_content_text 表**

> 含义: 文本类型的消息内容

| 字段       | 含义           |
| ------------- |:-------------:|
| _id      | 主键 |
| msg_id      | 属于哪条消息? |
| body      | 具体内容, TEXT sqlite3类型  |

**tb_chat_message_content_image 表**

> 含义: 图片类型的消息内容

| 字段       | 含义           |
| ------------- |:-------------:|
| _id      | 主键 |
| msg_id      | 属于哪条消息? |
| download_url      | 下载地址  |


还有其他 **tb_chat_message_content_XXXYYYZZZZ 表** ... 就不一一列举了

## SQL 语句怎么写?

<pre class="line-numbers" data-start="1" data-line=""><code class="language-sql">

SELECT
	chat_with,
    tb_conversation.type as conversation_type
	msg_id,
	MAX(timestamp) AS tp,
FROM
	tb_chat_message
JOIN tb_conversation ON tb_chat_message.chat_with = tb_conversation.chat_with
AND (
	tb_conversation.type = 0
	OR tb_conversation.type = 1
)
GROUP BY
	chat_with
ORDER BY
	tp

</code></pre>


**讲解**

1 先连接查询所有最近会话的消息

<pre class="line-numbers" data-start="8" data-line=""><code class="language-sql">

JOIN tb_conversation ON tb_chat_message.chat_with = tb_conversation.chat_with
AND (
	tb_conversation.type = 0
	OR tb_conversation.type = 1
)

</code></pre>

2 对上面的结果分组

<pre class="line-numbers" data-start="13" data-line=""><code class="language-sql">

GROUP BY
	chat_with

</code></pre>

3 分别为每一组选出消息时间最大的那条记录 (其实就是每个会话的最后一条消息)

<pre class="line-numbers" data-start="5" data-line=""><code class="language-sql">

MAX(timestamp) AS tp,

</code></pre>

4 排序

<pre class="line-numbers" data-start="15" data-line=""><code class="language-sql">

ORDER BY
	tp

</code></pre>

## 查询速度

**环境** :

- **tb_chat_message 表** 有 10 000 条数据
- **tb_conversation 表** 有 100 条数据
- 测试环境 : window7 配置一般吧...

**耗时** :  0.207 s

## 总结

上面的内容还算比较简单, 用过 Wechat 的人都知道真实情况还包含 `置顶`, `草稿`, `公众号`, `订阅号`...

这样子的排序规则可复杂了... ( ^_^ )/~~拜拜






