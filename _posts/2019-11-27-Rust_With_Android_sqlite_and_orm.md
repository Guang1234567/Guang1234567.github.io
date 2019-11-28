---
layout:       post
title:        "Rust 与 Android 之 sqlite3 and ORM"
subtitle:     ""
date:         2019-11-27 00:00:05
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
    -
---


> 此文章不允许转载, 违者必究...

> 目的:

**目录:**

* content
{:toc}


## 简概

解决 app Model 层的问题: 持久化数据到数据库

 [可执行的正常运行的例子](https://github.com/Guang1234567/rust_android_common_lib) 在这里.

## 开始

### 直接操作 sqlite

借助 `rusqlite` 库

**File: `Cargo.toml`** 

```toml
[package]
name = "rust_android_common_lib"
version = "0.1.0"
authors = ["Guang1234567 <abc@163.com>"]
edition = "2018"

# ...

[target.'cfg(target_os="android")'.dependencies]
rusqlite = {version = "0.20.0", features = ["bundled"]}

# ... 
[lib]
# 编译的动态库名字
name = "greetings"
# 编译类型 cdylib 指定为动态库
crate-type = ["cdylib", "staticlib"]
```
<br/>

---

如何使用?

**File: `sqlite.rs`** 

```rust
use std::path::Path;

use rusqlite::{Connection, params, Result, NO_PARAMS};
use rusqlite::types::ToSql;

use time::Timespec;

#[derive(Debug)]
struct Person {
    id: i32,
    name: String,
    time_created: Timespec,
    data: Option<Vec<u8>>,
}

pub struct SqliteHelper {}


impl SqliteHelper {
    pub fn open<P: AsRef<Path>>(path: P) -> Result<Connection> {
        Connection::open(path)
    }

    pub fn open_in_memory() -> Result<Connection> {
        Connection::open_in_memory()
    }

    pub fn write_sth_to_db(mut conn: Connection) -> Result<()> {

        let tx = conn.transaction()?;

        tx.execute(
            "CREATE TABLE IF NOT EXISTS person (
                  id              INTEGER PRIMARY KEY,
                  name            TEXT NOT NULL,
                  time_created    TEXT NOT NULL,
                  data            BLOB
                  )",
            NO_PARAMS,
        )?;
        let me = Person {
            id: 0,
            name: "Steven".to_string(),
            time_created: time::get_time(),
            data: None,
        };
        tx.execute(
            "INSERT INTO person (name, time_created, data)
                  VALUES (?1, ?2, ?3)",
            params![me.name, me.time_created, me.data],
        )?;

        tx.commit();

        let mut stmt = conn.prepare("SELECT id, name, time_created, data FROM person")?;
        let person_iter = stmt.query_map(params![], |row| {
            Ok(Person {
                id: row.get(0)?,
                name: row.get(1)?,
                time_created: row.get(2)?,
                data: row.get(3)?,
            })
        })?;

        for person in person_iter {
            warn!("Found person {:?}", person?);
        }
        Ok(())
    }
}
```
<br/>

---

**缺点**

- 随着业务的开展, 数据库的升级是必须的,  但 `rusqlite` 不提供升级数据库的接口, 需要额外写代码来升级.
- so 的体积变大, 因为把 sqlite3 静态编译进去了, 没有共用 android os 里的 `libsqlite3.so`  .
- ...


### 借助 ORM 框架 : diesel

diesel 提供了数据库升级迁移的功能,  提供一套不用直接面对 sql 的 DSL 接口 (算是有优点吧).

#### step1:  diesel 官方教程

https://diesel.rs/guides/getting-started/

#### step2:  移植到 android 上需要做的额外的事情

增加 `diesel_migrations = "1.4.0"` 依赖

```toml
[package]
name = "rust_android_common_lib"
version = "0.1.0"
authors = ["Guang1234567 <abc@163.com>"]
edition = "2018"

[target.'cfg(target_os="android")'.dependencies]
diesel = { version = "1.4.3", features = ["sqlite"] }
diesel_migrations = "1.4.0"
```

 通过调用
 
 宏
 
```rust
embed_migrations!("./migrations");
```

在 compile-time(编译时),把升级所用到的 sql 语句和升级相关的方法(`diesel_migrations` 库提供)编译到 rust 的二进制代码当中去. 如:

```rust
pub fn db_migrations(conn: &SqliteConnection) -> LibResult<()> {
    warn!(">>>>>>>---------------------  db_migrations  ---------------------");
    embed_migrations!("./migrations");
    let output = &mut MyLogger::new();
    let r = embedded_migrations::run_with_output(conn, output)?;
    output.flush();
    warn!("<<<<<<<---------------------  db_migrations  ---------------------");
    Ok(r)
}
```

另外, 上面代码把数据库升级迁移的 log 重定向到 `MyLogger`,  而 `MyLogger` 又重定向到 `Logcat`.

此时数据库会自动升级

#### step3: 代码演示

```rust
use std::env;
use std::io::Write;

use diesel::connection::SimpleConnection;
use diesel::prelude::*;
use diesel::SqliteConnection;

use crate::error::LibResult;
use crate::logger::MyLogger;

use super::models::{NewPost, Post};
use super::schema::posts::dsl::{posts as posts_table, published as posts_published, title as posts_title};

pub fn do_some_db_op(database_url: String) -> LibResult<()> {
    let conn = &establish_connection(database_url)?;

    db_migrations(conn)?;

    let inserted_rows = create_post(conn, "title001", "body001")?;

    publish_post(conn, 1)?;

    show_posts(conn)?;

    Ok(())
}

pub fn establish_connection(database_url: String) -> LibResult<SqliteConnection> {
    /*
    let database_url = env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");
    */
    Ok(SqliteConnection::establish(&database_url)?)
}

pub fn db_migrations(conn: &SqliteConnection) -> LibResult<()> {
    warn!(">>>>>>>---------------------  db_migrations  ---------------------");
    embed_migrations!("./migrations");
    let output = &mut MyLogger::new();
    let r = embedded_migrations::run_with_output(conn, output)?;
    output.flush();
    warn!("<<<<<<<---------------------  db_migrations  ---------------------");
    Ok(r)
}

pub fn show_posts(conn: &SqliteConnection) -> LibResult<Vec<Post>> {
    let posts: Vec<Post> = posts_table
        .filter(posts_published.eq(true))
        .limit(5)
        .load::<Post>(conn)?;
    //.expect("Error loading posts");

    error!("Displaying {} posts", posts.len());
    warn!("Displaying {} posts", posts.len());
    info!("Displaying {} posts", posts.len());
    debug!("Displaying {} posts", posts.len());
    trace!("Displaying {} posts", posts.len());
    for post in &posts {
        info!("{}", post.title);
        info!("----------\n");
        info!("{}", post.body);
    }

    Ok(posts)
}

pub fn create_post(conn: &SqliteConnection, title: &str, body: &str) -> LibResult<usize> {
    let new_post = NewPost { title, body };

    let inserted_rows = diesel::insert_into(posts_table)
        .values(&new_post)
        .execute(conn)?;
    //.expect("Error saving new post")

    info!("create_post  inserted_rows={}", inserted_rows);

    Ok(inserted_rows)
}

pub fn publish_post(conn: &SqliteConnection, id: i32) -> LibResult<Post> {
    let updated_rows = diesel::update(posts_table.find(id))
        .set(posts_published.eq(true))
        .execute(conn)?;
    //.unwrap_or_else(|_| panic!("Unable to find post {}", id));

    let post: Post = posts_table
        .find(id)
        .first(conn)?;
    //.unwrap_or_else(|_| panic!("Unable to find post {}", id));

    info!("Published post {}", post.title);

    Ok(post)
}

pub fn delete_post(conn: &SqliteConnection, pattern: String) -> LibResult<usize> {
    let pattern = format!("%{}%", pattern);

    let deleted_rows = diesel::delete(posts_table.filter(posts_title.like(pattern)))
        .execute(conn)?;
    //.expect("Error deleting posts");

    info!("Deleted {} posts", deleted_rows);

    Ok(deleted_rows)
}
```










