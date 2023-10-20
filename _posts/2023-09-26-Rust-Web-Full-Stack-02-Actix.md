---
layout: post
title: Rust Web 全栈开发教程-02-Actix 和 REST API
date: 2023-09-26 16:45:30.000000000 +09:00
categories: [Rust, Rust Web]
tags: [Rust, Rust Web]
---


# 2 Actix 框架尝鲜入门

## 2.1 添加 crate 依赖
* `actix-web`：即 `Actix` web 框架，版本是 `v3`
* `actix-rt`：即 `Actix` 异步的运行时，`actix-web v3 依赖 actix-rt 的版本是 1.1.1`


## 2.2 操作
* Step 1：创建 `workspace`，编辑 `workspace 的 Cargo.toml`

```rust
[workspace]
members = ["webservice"]
```

* Step 2：创建 `webservice` 
```shell
$ cargo new webservice
```

* Step 3：打开 `webservice` 中的 `Cargo.toml` 添加依赖

```rust
[package]
name = "webservice"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "3"
actix-rt = "1.1.1"

# 指定二进制名称
[[bin]]
name = "server1"

# [[bin]] 类似一个数组，可指定多个区域，这里还可以有个 server2
#[[bin]]
#name = "server2"
```

* Step 4：在 `webservice` 的 `src` 目录中添加 `bin` 目录，在 `bin` 目录中添加 `server1.rs` 文件，
1. `server1.rs` 中包含 `server1 二进制的 main 函数`

```
目录结构
webservice
 |-src
  |-bin
   |-server1.rs
```


* server1.rs

```rust
use actix_web::{web, App, HttpResponse, HttpServer, Responder};
use std::io;

// 配置 route
pub fn general_routes(cfg: &mut web::ServiceConfig) {
    /*
        设置路由 path 为 /health
        web::get() 表示使用 http get，
        to 中函数 health_check_handler 就是对应的 Handler
     */
    cfg.route("/health", web::get().to(health_check_handler));
}

// 配置 Handler

pub async fn health_check_handler() -> impl Responder {
    /*
        返回 Ok Response，并返回 json 为 Actix Web Service is running
        要求返回值实现 Responder trait
     */
    HttpResponse::Ok().json("Actix Web Service is running!")
}
 
// 实例化 HTTP server 并运行
#[actix_rt::main] // 用到 actix_rt 运行时
async fn main() -> io::Result<()> {
    // 构建 app，配置 route 路由，传入 general_routes 函数
    let app = move || App::new().configure(general_routes);

    // 运行 HTTP server
    // new 函数初始化一个 HttpServer，然后绑定到 3000 地址并运行
    HttpServer::new(app).bind("127.0.0.1:3000")?.run().await
}
```

* Step 5： 运行程序

在 `workspace` 根目录运行 

```shell
$ cargo run -p webservice --bin server1
```

* 或在 `webservice` 目录中

```shell
$ cargo run --bin server1
```

* 然后在浏览器中访问 `http://localhost:3000/health` 即可


## 2.3 基本组件

* `Actix HTTP Server` 它实现了 HTTP 协议，它负责应对请求的，它会开启多个线程来应对请求

![image](/assets/images/rust/web_server/actix.png)


## 2.4 `Actix` 的并发

### `Actix` 支持两类并发：
1. `异步 I/O`：给定的 `OS 原生线程` 在等待 `I/O` 时可以执行其他任务（例如监听网络连接）
2. `多线程并行`：默认情况下启动 `OS 原生线程` 的数量与系统逻辑 `CPU` 的数量是相同的



# 3 构建 REST API

## 3.1 添加 crate 依赖
* `serde`：即序列化和反序列化的库，版本是 `v1`
* `chrono`：用于使用时间相关的字段，版本是 `v0.4`


## 3.2 构建内容

![image](/assets/images/rust/web_server/rest_api.png)

1. 能添加课程，`POST:/courses`
2. 能获取课程，`GET:/courses/teacher_id/course_id`
3. 能获取老师教的所有课程，`GET:/courses/teacher_id`
4. 存储使用内存

* 相关文件
1. bin/teacher-service.rs
2. models.rs
3. state.rs：应用程序状态
4. routers.rs
5. handlers.rs


## 3.3 健康检查

* Step 1：修改 `webservice` 的 `toml`，添加 `teacher-service`，并添加 `default-run` 配置项

```rust
# 这是 webservice 的 toml
 
[package]
name = "webservice"
version = "0.1.0"
edition = "2021"

# 当运行 webservice 时，如果不指名二进制文件，那么首先运行 teacher-service
default-run = "teacher-service"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
actix-web = "3"
actix-rt = "1.1.1"

# 指定二进制名称
[[bin]]
name = "server1"

# [[bin]] 类似一个数组，可指定多个区域，这里还可以有个 teacher-service
#[[bin]]
#name = "teacher-service"
```

* Step 2：在 `bin` 目录创建 `teacher-service.rs`，在 `src` 目录下创建 `models.rs、state.rs、routers.rs、handlers.rs`


> 使用 `Actix` 框架，`state`（应用程序状态） 就可以被注入到请求它的 `Handler` 中，这些 `Handler` 就可以根据方法签名中的参数来访问这些 `state`
{: .prompt-info }


* Step 3：运行

```shell
// 运行 teacher-service，因为配置了 default-run 所以 cargo run 不指定二进制默认运行 teacher-service
$ cargo run -p webservice
```

可以再开一个命令行用 `curl` 进行验证，可 `curl` 多次，以便验证次数

```shell
$ curl localhost:3000/health
```

## 3.4 Post 资源

* Step 1：修改 `webservice` 的 `toml`，添加依赖库 `serde` 和 `chrono`

```rust
# 这是 webservice 的 toml
 
[package]
name = "webservice"
version = "0.1.0"
edition = "2021"

# 当运行 webservice 时，如果不指名二进制文件，那么首先运行 teacher-service
default-run = "teacher-service"

[dependencies]
actix-web = "3"
actix-rt = "1.1.1"
serde = { version = "1.0.132", features = ["derive"]}  // 新增
chrono = { version = "0.4.19", features = ["serde"]}   // 新增
```

其他代码

* state.rs

```rust
use std::sync::Mutex;
use super::models::Course;

pub struct AppState {
    // 响应字符串，这个字段共享于所有线程，初始化后它是一个不可变的
    pub health_check_response: String,
    /*
        也可以给每个线程共享，但它是可变的
        使用 Mutex 保证线程安全，即在修改数据前这个线程要先获取修改数据的控制权
     */
    pub visit_count: Mutex<u32>,
    
    pub courses: Mutex<Vec<Course>>,  // 新增字段
}
```

* routers.rs

```rust
pub fn course_routes(cfg: &mut web::ServiceConfig) {
    /*
        service 方法使用 scope 先定义了作用域

        /courses 就是这套的根路径
     */
    cfg
    .service(web::scope("/courses")
    .route("/", web::post()
    .to(new_course)));
}
```

* models.rs

```rust
use actix_web::web;
use chrono::NaiveDateTime;
use serde::{Deserialize, Serialize}; // 序列化，反序列化

// Clone 用于解决所有权相关问题
#[derive(Deserialize, Serialize, Debug, Clone)]
pub struct Course {
    pub teacher_id: usize,
    pub id: Option<usize>,
    pub name: String,
    pub time: Option<NaiveDateTime>, // 时间类型
}
 
// 实现 From 将 json 格式数据转为 Course
/*
    web::Json<T>、web::Data<T> 都属于叫数据提取器
    这里作用就是可以把 json 格式数据转为 Course 等特定类型的数据

*/
impl From<web::Json<Course>> for Course {
    fn from(course: web::Json<Course>) -> Self {
        Course {
            teacher_id: course.teacher_id,
            id: course.id,
            name: course.name.clone(),
            time: course.time,
        }
    } 
}
```

* handler.rs

```rust
use super::models::Course;
use chrono::Utc;

// 经测试，app_state 与 new_course 顺序可互换
pub async fn new_course(
    app_state: web::Data<AppState>,
    new_course: web::Json<Course>, 
) -> HttpResponse {
    println!("Received new course");

    let course_count = app_state
        .courses
        .lock()
        .unwrap()
        .clone()
        .into_iter()  // 变成便利器
        .filter(|course| course.teacher_id == new_course.teacher_id)
        .collect::<Vec<Course>>() // 变成 Vector
        .len();

    let new_course = Course {
        teacher_id: new_course.teacher_id, // 传进来的 id
        id: Some(course_count + 1),
        name: new_course.name.clone(),
        time: Some(Utc::now().naive_utc()), // 取当前时间
    };
    // 加入新课程到集合中
    app_state.courses.lock().unwrap().push(new_course);
    HttpResponse::Ok().json("Course added!")


}

#[cfg(test)]
mod tests {
    use super::*;
    use actix_web::http::StatusCode;
    use std::sync::Mutex;

    // 通常测试写个 test 就行了，但这里是 async 的所以需要用 actix_rt 异步运行时
    #[actix_rt::test]
    async fn post_course_test() {
        let course = web::Json(
            Course {
                teacher_id: 1,
                name: "Test Course".into(), // 用 to_string() 也行
                id: None,
                time: None,
            }
        );
        let app_state: web::Data<AppState> = web::Data::new(
            AppState {
                health_check_response: "".to_string(),
                visit_count: Mutex::new(0),
                courses: Mutex::new(vec![]),
            }
        );
        let resp = new_course(app_state, course).await;
        assert_eq!(resp.status(), StatusCode::OK);
    }
}

测试
$ cargo test -p webservice
```

* teacher-service.rs

```rust
#[actix_rt::main]
async fn main() -> io::Result<()> {
    let shared_data = web::Data::new(
        // 初始化 AppState
        AppState {
            health_check_response: "I'm OK.".to_string(),
            visit_count: Mutex::new(0),
            courses: Mutex::new(vec![]),
        }
    );

    // 这个闭包就是创建应用
    /*
        app_data(shared_data.clone()) 就是 把 shared_data 注册到 web 应用，
        这时就可以向 handler 中注入数据了

        configure(general_routes) 即配置它的路由
     */
    let app = move || {
        App::new()
        .app_data(shared_data.clone())
        .configure(general_routes)  // 
        .configure(course_routes)   // 新增，添加路由注册，general_routes 就是  routers 里的方法
    };

    HttpServer::new(app).bind("localhost:3000")?.run().await
}
```


* 测试

```shell
$ curl -X POST localhost:3000/courses/ -H "Content-Type: application/json" -d '{"teacher_id":1,"name":"First course"}'

# 返回 "Course added!"% 就可以了
```


## 3.5 Get 资源

* routers.rs

```rust
pub fn course_routes(cfg: &mut web::ServiceConfig) {
    /*
        service 方法使用 scope 先定义了作用域

        /courses 就是这套的根路径
     */
    cfg
    .service(web::scope("/courses")
    // POST localhost:3000/courses/
    .route("/", web::post().to(new_course))
    // GET localhost:3000/courses/{user_id}
    .route("/{user_id}", web::get().to(get_courses_for_teacher))); // 新增 GET 查询
}
```

* handlers.rs

```rust
// GET localhost:3000/courses/{user_id}
pub async fn get_courses_for_teacher(
    app_state: web::Data<AppState>,
    params: web::Path<(usize)>,
) -> HttpResponse {
    // Path 里是一个元组，元组就一个元素，类型是 usize
    let teacher_id: usize = params.0;

    let filtered_courses = app_state
        .courses
        .lock()
        .unwrap()
        .clone()
        .into_iter()
        .filter(|course| course.teacher_id == teacher_id) 
        .collect::<Vec<Course>>();

    if filtered_courses.len() > 0 {
        HttpResponse::Ok().json(filtered_courses)
    } else {
        HttpResponse::Ok().json("No courses found for teacher".to_string())
    }
}


// tests

    #[actix_rt::test]
    async fn get_all_course_success() {
        let app_state: web::Data<AppState> = web::Data::new(
            AppState {
                health_check_response: "".to_string(),
                visit_count: Mutex::new(0),
                courses: Mutex::new(vec![]),
            }
        );
        // Path::from 创建 id 为 1 的课程
        let teacher_id: web::Path<(usize)> = web::Path::from(1);
        let resp = get_courses_for_teacher(app_state, teacher_id).await;
        assert_eq!(resp.status(), StatusCode::OK);
    }
```

* 测试

```shell
# 先造几个数据
$ curl -X POST localhost:3000/courses/ -H "Content-Type: application/json" -d '{"teacher_id":1,"name":"First course"}'
$ curl -X POST localhost:3000/courses/ -H "Content-Type: application/json" -d '{"teacher_id":1,"name":"Second course"}'
$ curl -X POST localhost:3000/courses/ -H "Content-Type: application/json" -d '{"teacher_id":1,"name":"Third course"}'


# 访问数据
git:(master) ✗ curl localhost:3000/courses/1

# 返回下边 说明 OK
[{"teacher_id":1,"id":1,"name":"First course","time":"2023-10-17T07:43:29.087966"},{"teacher_id":1,"id":2,"name":"Second course","time":"2023-10-17T07:43:37.161619"},{"teacher_id":1,"id":3,"name":"Third course","time":"2023-10-17T07:43:43.974839"}]%
```

[项目地址](https://github.com/ZacharyWulven/Rust_Web_Full_Stack_Guide/tree/master/ws-02)

# 4 连接数据库

## 需要使用 `crate` 和数据库
* sqlx：用于连接数据库的 `crate`，版本 v0.5.10
* PostgreSQL：数据库


* Cargo.toml

```rust
[package]
name = "db"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
actix-rt = "2.6.0"
actix-web = "3.3.3"
chrono = {version = "0.4.19", features = ["serde"]}

# 用于设置环境变量
dotenv = "0.15.0"
openssl = {version = "0.10.38", features = ["vendored"]}
serde = {version = "1.0.134", features = ["derive"]}
sqlx = {version = "0.5.10", default_features = false, features = [
    "postgres",
    "runtime-tokio-rustls",
    "macros",
    "chrono",
]}
```

* main.rs

```rust
use chrono::NaiveDateTime;
use dotenv::dotenv;
use sqlx::postgres::PgPoolOptions;
use std::env;
use std::io;


#[derive(Debug)]
pub struct Course {
    pub id: i32,
    pub teacher_id: i32,
    pub name: String,
    pub time: Option<NaiveDateTime>,
}


#[actix_rt::main]
async fn main() -> io::Result<()> {
    // 读取 .env 的环境变量，返回 Result，为了防止警告这里调用了 ok()
    // 在生成环境不会使用 .env 设置环境变量
    // 调用 ok() 会把 Result 转为 Option 类型，即使失败了也会被忽略  
    dotenv().ok();

    
    let database_url = env::var("DATABASE_URL").expect("DATABASE_URL is not in .env file");

    let db_pool = PgPoolOptions::new()
        .connect(&database_url)
        .await
        .unwrap();

    let course_rows = sqlx::query!(
        r#"select id, teacher_id, name, time from course where id = $1"#,
        1,
    ).fetch_all(&db_pool)
    .await
    .unwrap();

    let mut courses_list = vec![];
    for row in course_rows {
        courses_list.push(
            Course {
                id: row.id,
                teacher_id: row.teacher_id,
                name: row.name,
                time: Some(chrono::NaiveDateTime::from(row.time.unwrap())),
            }
        )
    }
    println!("Courses = {:?}", courses_list);
    Ok(())
}
```

* db.sql

```rust
drop table if exists course;

create table course(
    id serial primary key,
    teacher_id INT not null,
    name varchar(140) not null,
    time TIMESTAMP default now()
);


insert into course(
    id, teacher_id, name, time)
values(1, 1, 'First course', '2022-01-12 05:40:00')
insert into course(
    id, teacher_id, name, time)
values(2, 1, 'Second course', '2022-01-15 08:40:00')
```

* .env

```
#                       用户名    密码                   数据库名称
DATABASE_URL=postgres://postgres:123456@127.0.0.1:5432/testdb
```

# 5 之前的 Web 项目使用数据库

[DB Ranking](https://db-engines.com/en/ranking)


[使用数据库仓库](https://github.com/ZacharyWulven/Rust_Web_Full_Stack_Guide/commit/8aa024d9e2b95e588c2a29660c0100661245ec79#diff-fc8bfafb0fad95bac0b95dbdbac05a3cfc88f93a1e434a48277483864caf3c91)

* state.rs

```rust
use std::sync::Mutex;
// use super::models::Course;
use sqlx::postgres::PgPool;

pub struct AppState {
    // 响应字符串，这个字段共享于所有线程，初始化后它是一个不可变的
    pub health_check_response: String,
    /*
        也可以给每个线程共享，但它是可变的
        使用 Mutex 保证线程安全，即在修改数据前这个线程要先获取修改数据的控制权
     */
    pub visit_count: Mutex<u32>,
    
    // pub courses: Mutex<Vec<Course>>,
    pub db: PgPool, // 使用数据库新增
}
```


* teacher-service.rs

```rust
use actix_web::{web, App, HttpServer};
use std::io;
use std::sync::Mutex;
use dotenv::dotenv;
use std::env;
use sqlx::postgres::PgPoolOptions; // 数据库的连接池


// 定义模块指明路径, 声明模块
#[path = "../handlers.rs"]
mod handlers;

#[path = "../routers.rs"]
mod routers;

#[path = "../state.rs"]
mod state;

#[path = "../models.rs"]
mod models;

#[path = "../db_access.rs"]
mod db_access;


// 引入 routers 模块所有内容
use routers::*;
use state::AppState;


#[actix_rt::main]
async fn main() -> io::Result<()> {

    dotenv().ok();

    let database_url = env::var("DATABASE_URL")
        .expect("DATABASE_URL is not setup");

    let db_pool = PgPoolOptions::new()
        .connect(&database_url)
        .await
        .unwrap();

    let shared_data = web::Data::new(
        // 初始化 AppState
        AppState {
            health_check_response: "I'm OK.".to_string(),
            visit_count: Mutex::new(0),
            db: db_pool,       // 使用数据库改动
            // courses: Mutex::new(vec![]),
        }
    );

    // 这个闭包就是创建应用
    /*
        app_data(shared_data.clone()) 就是 把 shared_data 注册到 web 应用，
        这时就可以向 handler 中注入数据了

        configure(general_routes) 即配置它的路由
     */
    let app = move || {
        App::new()
        .app_data(shared_data.clone())
        .configure(general_routes)  // 添加路由注册，general_routes 就是  routers 里的方法
        .configure(course_routes)
    };

    HttpServer::new(app).bind("localhost:3003")?.run().await
}
```

* db_access.rs

```rust
use super::models::*;
use chrono::NaiveDateTime;
use sqlx::postgres::PgPool;

/*
    本函数用于读取老师的课
    PgPool 就是数据库连接池

*/
pub async fn get_courses_for_teacher_db(pool: &PgPool, teacher_id: i32) -> Vec<Course> {
    // query! 就是 format SQL 语句
    // SQL 语句涉及多行，前边加 r 即可写多行, 格式 r#"{ SQL query content}"#
    let rows = sqlx::query!(
        r#"SELECT id, teacher_id, name, time
        FROM course
        WHERE teacher_id = $1"#, 
        teacher_id
    )
    .fetch_all(pool)
    .await
    .unwrap(); 

    rows.iter()
        .map(|r| Course {
            id: Some(r.id),
            teacher_id: r.teacher_id,
            name: r.name.clone(),
            time: Some(NaiveDateTime::from(r.time.unwrap())),
        })
        .collect()
}

pub async fn get_course_detail_db(pool: &PgPool, teacher_id: i32, course_id: i32) -> Course {
    let row = sqlx::query!(
        r#"SELECT id, teacher_id, name, time
        FROM course
        WHERE teacher_id = $1 and id = $2"#,
        teacher_id,
        course_id,
    )
    .fetch_one(pool)
    .await
    .unwrap(); 

    Course {
        id: Some(row.id),
        teacher_id: row.teacher_id,
        name: row.name.clone(),
        time: Some(NaiveDateTime::from(row.time.unwrap())),
    }

}

pub async fn post_new_course_db(pool: &PgPool, new_course: Course) -> Course {
    let row = sqlx::query!(
        // time 就不写了 因为其在数据库有默认值 就是调用 now() 函数
        r#"INSERT INTO course (id, teacher_id, name)
        VALUES ($1, $2, $3)
        RETURNING id, teacher_id, name, time"#,
        new_course.id,
        new_course.teacher_id,
        new_course.name,
    )
    .fetch_one(pool)
    .await
    .unwrap(); 

    Course {
        id: Some(row.id),
        teacher_id: row.teacher_id,
        name: row.name.clone(),
        time: Some(NaiveDateTime::from(row.time.unwrap())),
    }

} 
```

* handler.rs

```rust
use super::state::AppState;
use actix_web::{web, HttpResponse};
use super::db_access::*;

// 任何数据在 Actix 中注册后，在 handler 中就可以将它们注入，形式是 web::Data<AppState>
pub async fn health_check_handler(app_state: web::Data<AppState>) -> HttpResponse {
    // 这里可直接访问，但类型还是 web::Data<AppState>
    let health_check_response = &app_state.health_check_response;
    // 访问 visit_count 前先调用 lock() 防止其他线程更新这个值
    let mut visit_count = app_state.visit_count.lock().unwrap();
    let response = format!("{} {} times", health_check_response, visit_count);
    // 更新数据，这时已经上锁了，当走完这个函数锁才解开
    *visit_count += 1;
    HttpResponse::Ok().json(&response)
}

use super::models::Course;
// use chrono::Utc;

// 经测试，app_state 与 new_course 顺序可互换
pub async fn new_course(
    app_state: web::Data<AppState>,
    new_course: web::Json<Course>, 
) -> HttpResponse {

    // new_course.into() 转为 Course 类型
    let course = post_new_course_db(&app_state.db, new_course.into()).await;
    HttpResponse::Ok().json(course)
}

// GET localhost:3000/courses/{teacher_id}
pub async fn get_courses_for_teacher(
    app_state: web::Data<AppState>,
    params: web::Path<(usize,)>, // 这里应该是个元组，即 (usize,)
) -> HttpResponse {

    // 尝试转换 params.0 为 i32
    let teacher_id = i32::try_from(params.0).unwrap();
    let courses = get_courses_for_teacher_db(&app_state.db, teacher_id).await;
    HttpResponse::Ok().json(courses)

}

// GET localhost:3000/courses/{user_id}/{course_id}
// Path 上参数有俩，即 user_id 和 course_id
pub async fn get_course_detail(
    app_state: web::Data<AppState>,
    params: web::Path<(usize, usize)>,
) -> HttpResponse {

    let teacher_id = i32::try_from(params.0).unwrap();
    let course_id = i32::try_from(params.1).unwrap();
    let course = get_course_detail_db(&app_state.db, teacher_id, course_id).await;
    HttpResponse::Ok().json(course)

}


#[cfg(test)]
mod tests {
    use super::*;
    use actix_web::http::StatusCode;
    use std::sync::Mutex;
    // use chrono::NaiveDateTime;
    use dotenv::dotenv;
    use sqlx::postgres::PgPoolOptions;
    use std::env;

    // 通常测试写个 test 就行了，但这里是 async 的所以需要用 actix_rt 异步运行时
    #[actix_rt::test]
    async fn post_course_test() {
        dotenv().ok();

        let db_url = env::var("DATABASE_URL").expect("DATABASE_URL is not setup");
        let db_pool = PgPoolOptions::new()
            .connect(&db_url).await.unwrap();

        let app_state: web::Data<AppState> = web::Data::new(
            AppState {
                health_check_response: "".to_string(),
                visit_count: Mutex::new(0),
                db: db_pool,
            }
        );

        // 本测试只能跑一次，因为跑完数据库里会有 id=3 的条目
        let course = web::Json(
            Course {
                teacher_id: 1,
                name: "Test Course".into(), // 用 to_string() 也行
                id: Some(3), // serial 类型，需要赋一个值
                time: None,
            }
        );

        let resp = new_course(app_state, course).await;
        assert_eq!(resp.status(), StatusCode::OK);
    }

    #[actix_rt::test]
    async fn get_all_course_success() {
        dotenv().ok();
        let db_url = env::var("DATABASE_URL").expect("DATABASE_URL is not setup");
        let db_pool = PgPoolOptions::new()
            .connect(&db_url).await.unwrap();


        let app_state: web::Data<AppState> = web::Data::new(
            AppState {
                health_check_response: "".to_string(),
                visit_count: Mutex::new(0),
                db: db_pool,
            }
        );
        // Path::from 创建 id 为 1 的课程
        let teacher_id: web::Path<(usize,)> = web::Path::from((1,));
        let resp = get_courses_for_teacher(app_state, teacher_id).await;
        assert_eq!(resp.status(), StatusCode::OK);
    }

    #[actix_rt::test]
    async fn get_one_course_success() {
        dotenv().ok();
        let db_url = env::var("DATABASE_URL").expect("DATABASE_URL is not setup");
        let db_pool = PgPoolOptions::new()
            .connect(&db_url).await.unwrap();

        let app_state: web::Data<AppState> = web::Data::new(
            AppState {
                health_check_response: "".to_string(),
                visit_count: Mutex::new(0),
                db: db_pool,
            }
        );
        let params: web::Path<(usize, usize)> = web::Path::from((1,1));
        let resp = get_course_detail(app_state, params).await;
        assert_eq!(resp.status(), StatusCode::OK);
    }
}
```

* models.rs

```rust
use actix_web::web;
use chrono::NaiveDateTime;
use serde::{Deserialize, Serialize}; // 序列化，反序列化

// Clone 用于解决所有权相关问题
#[derive(Deserialize, Serialize, Debug, Clone)]
pub struct Course {
    pub teacher_id: i32,
    pub id: Option<i32>,
    pub name: String,
    pub time: Option<NaiveDateTime>, // 时间类型
}
 
// 实现 From 将 json 格式数据转为 Course
/*
    web::Json<T>、web::Data<T> 都属于叫数据提取器
    这里作用就是可以把 json 格式数据转为 Course 等特定类型的数据
*/
impl From<web::Json<Course>> for Course {
    fn from(course: web::Json<Course>) -> Self {
        Course {
            teacher_id: course.teacher_id,
            id: course.id,
            name: course.name.clone(),
            time: course.time,
        }
    } 
}
```

* routers.rs

```rust
use super::handlers::*;
use actix_web::web;

pub fn general_routes(cfg: &mut web::ServiceConfig) {
    cfg.route("/health", web::get().to(health_check_handler));
}

pub fn course_routes(cfg: &mut web::ServiceConfig) {
    /*
        service 方法使用 scope 先定义了作用域

        /courses 就是这套的根路径
     */
    cfg
    .service(web::scope("/courses")
    // POST localhost:3000/courses/
    .route("/", web::post().to(new_course))
    // GET localhost:3000/courses/{user_id}
    .route("/{user_id}", web::get().to(get_courses_for_teacher))
    // GET localhost:3000/courses/{user_id}/{course_id}
    .route("/{user_id}/{course_id}", web::get().to(get_course_detail)));
}
```

* 相关命令

```shell
$ cargo check --bin teacher-service

// for test
$ cargo test --bin teacher-service
```

> `.env` 文件要放到与 `Cargo.toml` 同级目录下或放到 `workspace` 根目录下
{: .prompt-info }

* db.sql

```
drop table if exists course;

create table course(
    id serial primary key,
    teacher_id INT not null,
    name varchar(140) not null,
    time TIMESTAMP default now()
);


insert into course(
    id, teacher_id, name, time)
values(1, 1, 'First course', '2022-01-12 05:40:00')
insert into course(
    id, teacher_id, name, time)
values(2, 1, 'Second course', '2022-01-15 08:40:00')
```

* 测试运行
1. 浏览器输入 `http://localhost:3003/courses/1`，`http://localhost:3003/courses/1/2`

2. POST 添加课程，然后 `http://localhost:3003/courses/1/7`

```shell
curl -X POST localhost:3003/courses/ \
-H "Content-Type: application/json" \
-d '{"teacher_id":1, "id":7, "name":"Tom"}'
```

> 如果出现 `Error: Os { code: 48, kind: AddrInUse, message: "Address already in use" }`，可以换个 `bind` 的端口号即可
{: .prompt-info }
