---
layout: post
title: Rust Web 全栈开发教程-03-错误处理
date: 2023-09-28 16:45:30.000000000 +09:00
categories: [Rust, Rust Web]
tags: [Rust, Rust Web]
---


# 6 Web Service 中的错误处理

* 在 `Actix` 中有统一的错误处理机制 
1. 即将数据库错误、序列化错误等错误转为自定义的错误类型，再转为有意义的 `HTTP` 响应信息发给客户端

![image](/assets/images/rust/web_server/actix_err.png)


## `Actix-Web` 的错误处理

* 编程语言常用的两种错误处理方式：
1. 异常（比如 JAVA）
2. 返回值（比如 Go、Rust）


* Rust 希望开发者能显示的处理错误，因此可能出错的函数返回 `Result` 这个枚举类型

```rust
enum Result<T,E> {
  Ok(T), // 如果这个函数没有错误就返回 Ok，正确要返回的值就是 T
  Err(E),
}
```

* 例子：

```rust
use core::num;
use std::num::ParseIntError;

fn main() {
    // let result = square("25");
    let result = square("Rt");

    println!("{:?}", result);
}

// 这就是一个可能出错的函数
fn square(val: &str) -> Result<i32, ParseIntError> {
    match val.parse::<i32>() {
        // 能成功解析成一个整数，就返回这个整数的平方
        Ok(num) => Ok(num.pow(2)),
        Err(e) => Err(e),
    }
}
```


## `?` 运算符
* 在某个函数中使用 `?` 运算符，该运算符就会尝试从 `Result` 中获取值：
1. 如果没有获取成功，它就会接收 Error，中止函数执行，并把错误传播到调用该函数的函数


```rust
fn square(val: &str) -> Result<i32, ParseIntError> {
    /*
        如果成功获取 i32，则 num 的值就是其值，
        否则此函数直接返回了，返回的就是 Err(e)
     */
    let num = val.parse::<i32>()?;
    Ok(num ^ 2)
}
```


> 把 `?` 作用于 `Result`。如果结果是 `Ok`，`Ok` 中的值就是表达式的结果，然后继续执行。如果结果是 `Err`，则 `Err` 就是整个函数的返回值，就类似使用了 `return`
{: .prompt-info }


## 自定义错误类型
* 项目中产生的错误肯定不是同一类型的错误，是多种类型的，这里解决思路就是创建一个自定义错误类型，例如枚举或结构体

```rust
#[derive(Debug)]
pub enum MyError {
  ParseError,
  IOError,
}
```

### `Actix-Web` 也使用同样思路，把错误转化为 `HTTP Response`
* `Actix-Web` 定义了一个通用的错误类型 `actix_web::error::Error`（是个 `struct`）
1. 它实现了标准库的 `std::error::Error` 这个 trait


> 任何实现了标准库 `Error trait` 的类型，都可以通过 `?` 运算符转化为 `Actix 的 Error` 类型，即转化为 `actix_web::error::Error` 类型，`Actix 的 Error` 类型会自动转化为 `HTTP Response` 返回给客户端
{: .prompt-info }


* `Actix-Web` 中定义了 `ResponseError trait`：即任何实现了该 `trait` 的错误均可以转化为 `HTTP Response` 消息
* `Actix-Web` 框架中对一些常见的错误有一些内置的实现，这些错误会自动转为 `HTTP Response`，例如
1. Rust 标准 `I/O` 错误
2. `Serde` 错误（序列化、反序列化错误）
3. `Web` 错误：例如 `ProtocolError、Utf8Error、ParseError` 等等

* 而其他错误类型（内置实现不包含的类型）：就需要自定义实现错误到 `HTTP Response` 的转换


## 创建自定义的错误处理器
1. 创建一个自定义的错误类型，并让其实现 `From trait`，用于将其他错误类型转换为该类型
2. 为自定义错误类型实现 `ResponseError trait`，这样就可以作为响应返回回去了
3. 在 `handler` 中返回自定义错误类型
4. 然后 `Actix` 会把这个错误转化为 `HTTP` 响应



* errors.rs

```rust
use actix_web::{error, http::StatusCode, HttpResponse, Result};
use serde::Serialize;
use sqlx::error::Error as SQLxError;
use std::fmt;

#[derive(Debug, Serialize)]
pub enum MyError {
    DBError(String),
    ActixError(String),
    NotFound(String),
}

#[derive(Debug, Serialize)]
pub struct MyErrorResponse {
    error_message: String,
}

impl MyError {
    fn error_response(&self) -> String {
        match self {
            MyError::DBError(msg) => {
                println!("Database error occurred: {:?}", msg);
                "Database error".into()
            }
            MyError::ActixError(msg) => {
                println!("Server error occurred: {:?}", msg);
                "Internal server error".into()
            }
            MyError::NotFound(msg) => {
                println!("Not found error occurred: {:?}", msg);
                msg.into()
            }
        }
    }
}

// actix_web::error
// 实现这个 trait 后当错误发生时 actix 就可以把 MyError 中的错误自动转为 HttpResponse
// 实现 error::ResponseError trait 时要求必须实现 Debug 和 Display 这俩 trait
// Display 需要自己实现
impl error::ResponseError for MyError {
    fn status_code(&self) -> StatusCode {
        match self {
            MyError::DBError(msg) | MyError::ActixError(msg) => StatusCode::INTERNAL_SERVER_ERROR,
            MyError::NotFound(msg) => StatusCode::NOT_FOUND,
        }
    }

    fn error_response(&self) -> HttpResponse {
        HttpResponse::build(self.status_code()).json(MyErrorResponse {
            error_message: self.error_response(),
        })
    }
}

impl fmt::Display for MyError {
    fn fmt(&self, f: &mut fmt::Formatter) -> Result<(), fmt::Error> {
        write!(f, "{}", self)
    }
}

// 可使 actix_web::error::Error 转为 MyError
impl From<actix_web::error::Error> for MyError {
    fn from(err: actix_web::error::Error) -> Self {
        MyError::ActixError(err.to_string())
    }
}

// 可使 SQLxError 转为 MyError
impl From<SQLxError> for MyError {
    fn from(err: SQLxError) -> Self {
        MyError::DBError(err.to_string())
    }
}
```


* db_access.rs

```rust
use super::models::*;
use chrono::NaiveDateTime;
use sqlx::postgres::PgPool;
use super::errors::MyError;

/*
    本函数用于读取老师的课
    PgPool 就是数据库连接池

*/
pub async fn get_courses_for_teacher_db(pool: &PgPool, teacher_id: i32) -> Result<Vec<Course>, MyError> {
    // query! 就是 format SQL 语句
    // SQL 语句涉及多行，前边加 r 即可写多行, 格式 r#"{ SQL query content}"#
    let rows = sqlx::query!(
        r#"SELECT id, teacher_id, name, time
        FROM course
        WHERE teacher_id = $1"#, 
        teacher_id
    )
    .fetch_all(pool)
    .await?;
    // 改为使用 ?，如遇到错误就返回 MyError
    // .unwrap();

    // 这里要声明一下 Vec<Course>
    let courses:Vec<Course> = rows.iter()
        .map(|r| Course {
            id: Some(r.id),
            teacher_id: r.teacher_id,
            name: r.name.clone(),
            time: Some(NaiveDateTime::from(r.time.unwrap())),
        })
        .collect();
    
    match courses.len() {
        0 => Err(MyError::NotFound("Courses not found for teacher".into())),
        _ => Ok(courses),
    }

}
 
pub async fn get_course_detail_db(pool: &PgPool, teacher_id: i32, course_id: i32) -> Result<Course, MyError> {
    let row = sqlx::query!(
        r#"SELECT id, teacher_id, name, time
        FROM course
        WHERE teacher_id = $1 and id = $2"#,
        teacher_id,
        course_id,
    )
    .fetch_one(pool)
    .await;
    // .unwrap(); 

    if let Ok(row) = row {
        Ok(
            Course {
                id: Some(row.id),
                teacher_id: row.teacher_id,
                name: row.name.clone(),
                time: Some(NaiveDateTime::from(row.time.unwrap())),
            }
        ) 
    } else {
        Err(MyError::NotFound("Course Id not found".into()))
    }
}

pub async fn post_new_course_db(pool: &PgPool, new_course: Course) -> Result<Course, MyError> {
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
    .await?;
    // .unwrap(); 

    Ok(Course {
        id: Some(row.id),
        teacher_id: row.teacher_id,
        name: row.name.clone(),
        time: Some(NaiveDateTime::from(row.time.unwrap())),
    })

} 
```


* handlers.rs

```rust
use crate::errors::MyError;

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
) -> Result<HttpResponse, MyError> {
    // println!("Received new course");

    // let course_count = app_state
    //     .courses
    //     .lock()
    //     .unwrap()
    //     .clone()
    //     .into_iter()  // 变成便利器
    //     .filter(|course| course.teacher_id == new_course.teacher_id)
    //     .collect::<Vec<Course>>() // 变成 Vector
    //     .len();

    // let new_course = Course {
    //     teacher_id: new_course.teacher_id, // 传进来的 id
    //     id: Some(course_count + 1),
    //     name: new_course.name.clone(),
    //     time: Some(Utc::now().naive_utc()), // 取当前时间
    // };
    // // 加入新课程到集合中
    // app_state.courses.lock().unwrap().push(new_course);

    // new_course.into() 转为 Course 类型
    post_new_course_db(&app_state.db, new_course.into())
    .await
    .map(|course| HttpResponse::Ok().json(course))
    
}

// GET localhost:3000/courses/{teacher_id}
pub async fn get_courses_for_teacher(
    app_state: web::Data<AppState>,
    params: web::Path<(usize,)>, // 这里应该是个元组，即 (usize,)
) -> Result<HttpResponse, MyError> {
    // Path 里是一个元组，元组就一个元素，类型是 usize
    // let teacher_id: usize = params.0;

    // let filtered_courses = app_state
    //     .courses
    //     .lock()
    //     .unwrap()
    //     .clone()
    //     .into_iter()
    //     .filter(|course| course.teacher_id == teacher_id) 
    //     .collect::<Vec<Course>>();

    // if filtered_courses.len() > 0 {
    //     HttpResponse::Ok().json(filtered_courses)
    // } else {
    //     HttpResponse::Ok().json("No courses found for teacher".to_string())
    // }
    // 尝试转换 params.0 为 i32
    let teacher_id = i32::try_from(params.0).unwrap();
    get_courses_for_teacher_db(&app_state.db, teacher_id)
    .await
    .map(|courses|
        HttpResponse::Ok().json(courses)
    )
    // 上边发送错误的话 类型就是 MyError
    // 由于 MyError 实现了 error::ResponseError，所以 actix 会自动转为 HttpResponse 返回给用户

}

// GET localhost:3000/courses/{user_id}/{course_id}
// Path 上参数有俩，即 user_id 和 course_id
pub async fn get_course_detail(
    app_state: web::Data<AppState>,
    params: web::Path<(usize, usize)>,
) -> Result<HttpResponse, MyError> {
    // let (teacher_id, course_id) = params.0;
    // let selected_course = app_state
    //     .courses
    //     .lock()
    //     .unwrap()
    //     .clone()
    //     .into_iter()
    //     .find(|x| x.teacher_id == teacher_id && x.id == Some(course_id))
    //     // 调用 ok_or 将 Option<T> 类型转为 Result<T,E> 类型
    //     // 如果 Option<T> 中有值就返回 Ok，否则返回 Err
    //     .ok_or("Course not found"); 

    // if let Ok(course) = selected_course {
    //     HttpResponse::Ok().json(course)
    // } else {
    //     HttpResponse::Ok().json("Course not found".to_string())
    // } 

    let teacher_id = i32::try_from(params.0).unwrap();
    let course_id = i32::try_from(params.1).unwrap();
    get_course_detail_db(&app_state.db, teacher_id, course_id)
    .await
    .map(|course| 
        HttpResponse::Ok().json(course)
    )
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

    // 用于忽略这个测试
    #[ignore]
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
                id: Some(4), // serial 类型，需要赋一个值
                time: None,
            }
        );

        let resp = new_course(app_state, course).await.unwrap();
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
        let resp = get_courses_for_teacher(app_state, teacher_id).await.unwrap();
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
        let resp = get_course_detail(app_state, params).await.unwrap();
        assert_eq!(resp.status(), StatusCode::OK);
    }
}
```


* 运行测试

```shell
// 测试 get_all_course_success 函数
$ cargo test --bin teacher-service get_all_course_success
```

[错误处理 Demo](https://github.com/ZacharyWulven/Rust_Web_Full_Stack_Guide/commit/d47962bbdc4f252d8247de5ea1dc0fe44c1397b9)
