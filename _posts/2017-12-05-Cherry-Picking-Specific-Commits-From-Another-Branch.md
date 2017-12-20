---
layout:       post
title:        Cherry Picking Specific Commits From Another Branch
subtitle:     "git git git git baby baby 😆"
date:         2017-12-05 07:07:07
author:       "Guang1234567"
header-img:   "img/post-bg-git.jpg"
header-mask:  0.3
catalog:      true
multilingual: false
categories: 
    - cvs
    - git
tags:
    - git
---

> 此文章不允许转载, 违者必究...

> 目的: 如何使用 `git rebase --onto` 命令来将某一分支的某些相连的 Commit 合并到指定的另外一个分支, 避免用 cherry-pick 一个个合并.

**目录:**

* content
{:toc}

## Why 写这篇文章?

之前学习 git 官方教程的 [3.6 Git 分支 - 变基](https://git-scm.com/book/zh/v2/Git-%E5%88%86%E6%94%AF-%E5%8F%98%E5%9F%BA) 章节时,

对下面栏目的 `git rebase --onto` 到底能用来干什么感到有点疑惑:

![android-permission.svg]({{ site.baseurl }}/img/in-post/2017-12-05-Cherry-Picking-Specific-Commits-From-Another-Branch/git-rebase-onto.png)


接下来的某天看到这篇文章

> (原文) https://www.devroom.io/2010/06/10/cherry-picking-specific-commits-from-another-branch/
>
> (中文翻译) http://blog.csdn.net/ybdesire/article/details/42145597

才明白 `git rebase --onto` 的其中一个使用场景(可以这么玩) : 用来合并一连串的 commit 到另外一个分支.


## 用个例子来讲解

```
  ... - dd2e86 - 946992 -9143a9 - a6fd86 - 5a6057   [branch_dev_产品A]
                \
                76cada - 62ecb3 - b886a0 - 977f6f02   [branch_dev_产品B]
                ↑______________________↑
                             |
                          b1 功能
```

该公司有2个产品 `branch_dev_产品A` 和 `branch_dev_产品B`.
某一天有个大客户觉得 `branch_dev_产品B` 的 `b1 功能` 做的不错, 出价1个亿想买一个具有 `b1 功能` 的 `branch_dev_产品A`,
于是我定了个小目标...

>注:  `b1 功能` 由 `76cada - 62ecb3 - b886a0` 这 3 个 commit 组成.


### 步骤: 

#### STEP 1)

<script type="syntaxhighlighter" class="brush: bash; gutter: true; ruler: true; first-line: 1; highlight: [2]"><![CDATA[

Administrator@Guang123456-PC [12:19:35] [/d/my_blog] [branch_dev_产品B *]
-> % git checkout -b "branch_one_Million" b886a0

]]></script>

此时 git 仓库是这样的:

```
                                         [branch_dev_产品A]        
                                              |
                                              ↓
  ... - dd2e86 - 946992 -9143a9 - a6fd86 - 5a6057  
                \
                76cada - 62ecb3 - b886a0 - 977f6f02
                                    ↑        ↑
                                    |        |
                                    |    [branch_dev_产品B]
                                    |
                        [branch_one_Million]   
```

#### STEP 2)

<script type="syntaxhighlighter" class="brush: bash; gutter: true; ruler: true; first-line: 1; highlight: [2]"><![CDATA[

Administrator@Guang123456-PC [12:19:35] [/d/my_blog] [branch_one_Million *]
-> % git rebase --onto master 76cada^  

]]></script>

此时 git 仓库是这样的:

```
                                    [branch_dev_产品A]       [branch_one_Million]
                                              |                           |
                                              ↓                           ↓
  ... - dd2e86 - 946992 -9143a9 - a6fd86 - 5a6057 - 76cada' - 62ecb3' - b886a0'   
                \
                76cada - 62ecb3 - b886a0 - 977f6f02
                                              ↑
                                              |
                                        [branch_dev_产品B]
```

#### STEP 3)

<script type="syntaxhighlighter" class="brush: bash; gutter: true; ruler: true; first-line: 1; highlight: [2,6]"><![CDATA[

Administrator@Guang123456-PC [12:19:35] [/d/my_blog] [branch_one_Million *]
-> % git checkout branch_dev_产品A


Administrator@Guang123456-PC [12:19:35] [/d/my_blog] [branch_dev_产品A *]
-> % git merge branch_one_Million

]]></script>

此时 git 仓库是这样的:

```
                                                                 [branch_dev_产品A]
                                                                          |
                                                                 [branch_one_Million]
                                                                          |
                                                                          ↓
  ... - dd2e86 - 946992 -9143a9 - a6fd86 - 5a6057 - 76cada' - 62ecb3' - b886a0'                
                \
                76cada - 62ecb3 - b886a0 - 977f6f02
                                              ↑
                                              |
                                        [branch_dev_产品B]
```

<script type="syntaxhighlighter" class="brush: bash; gutter: true; ruler: true; first-line: 1; highlight: [3]"><![CDATA[

// branch_one_Million 完成任务, 删掉
Administrator@Guang123456-PC [12:19:35] [/d/my_blog] [branch_dev_产品A *]
-> % git branch -d branch_one_Million

]]></script>

