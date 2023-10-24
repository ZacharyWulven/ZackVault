---
layout: post
title: Rust Web 全栈开发教程-04-Refactor
date: 2023-10-01 16:45:30.000000000 +09:00
categories: [Rust, Rust Web]
tags: [Rust, Rust Web]
---


# 7 添加功能和重构

## 7.1 现状


![image](/assets/images/rust/web_server/refactor.png)


## 7.2 重构代码

* 重构后目录结构

![image](/assets/images/rust/web_server/web_server_dir.png)


* src/bin/teacher-service.rs

```rust
use actix_web::{web, App, HttpServer};
use std::io;
use std::sync::Mutex;
use dotenv::dotenv;
use std::env;
use sqlx::postgres::PgPoolOptions; // 数据库的连接池


// 定义模块指明路径, 声明模块
#[path = "../handlers/mod.rs"]
mod handlers;

#[path = "../routers.rs"]
mod routers;

#[path = "../state.rs"]
mod state;

#[path = "../models/mod.rs"]
mod models;

#[path = "../dbaccess/mod.rs"]
mod dbaccess;

#[path = "../errors.rs"]
mod errors;

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


* src/dbaccess/mod.rs

```rust
pub mod course;
```

* src/dbaccess/course.rs

```rust
// use std::error::Error;
use crate::models::course::{Course, UpdateCourse, CreateCourse};
// use chrono::NaiveDateTime;
use sqlx::postgres::PgPool;
use crate::errors::MyError;

/*
    本函数用于读取老师的课
    PgPool 就是数据库连接池

*/
pub async fn get_courses_for_teacher_db(pool: &PgPool, teacher_id: i32) -> Result<Vec<Course>, MyError> {
    // query! 就是 format SQL 语句
    // SQL 语句涉及多行，前边加 r 即可写多行, 格式 r#"{ SQL query content}"#

    // query_as! 可将结果转为 Vec<Course>, 第一个参数就是 Course，
    let rows: Vec<Course> = sqlx::query_as!(
        Course,
        r#"SELECT * FROM course WHERE teacher_id = $1"#, 
        teacher_id
    )
    .fetch_all(pool)
    .await?;
    // 改为使用 ?，如遇到错误就返回 MyError
    // .unwrap();

    Ok(rows)
}
 
pub async fn get_course_detail_db(pool: &PgPool, teacher_id: i32, course_id: i32) -> Result<Course, MyError> {
    let row = sqlx::query_as!(
        Course,
        r#"SELECT * FROM course WHERE teacher_id = $1 and id = $2"#,
        teacher_id,
        course_id,
    )
    // fetch_optional 表示有可能查询到结果 有可能查询不到
    .fetch_optional(pool)
    .await?;
    // .unwrap(); 

    // 如果查询到结果了 就返回 Ok
    if let Some(course) = row {
        Ok(course) 
    } else {
        Err(MyError::NotFound("Course Id not found".into()))
    }
}

pub async fn post_new_course_db(pool: &PgPool, new_course: CreateCourse) -> Result<Course, MyError> {
    let course = sqlx::query_as!(
        Course,
        // time 就不写了 因为其在数据库有默认值 就是调用 now() 函数
        r#"INSERT INTO course (teacher_id, name, description, format, structure, duration, price, language, level)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
        RETURNING id, teacher_id, name, time, description, format, structure, duration, price, language, level"#,
        new_course.teacher_id,
        new_course.name,
        new_course.description,
        new_course.format,
        new_course.structure,
        new_course.duration,
        new_course.price,
        new_course.language,
        new_course.level,
    )
    .fetch_one(pool)
    .await?;
    // .unwrap(); 

    Ok(course)
} 

pub async fn delete_course_db(pool: &PgPool, teacher_id: i32, id: i32) -> Result<String, MyError> {
    let course = sqlx::query!(
        "DELETE FROM course where teacher_id = $1 and id = $2",
        teacher_id,
        id,
    )
    .execute(pool)
    .await?;

    Ok(format!("Deleted {:?} record", course))
}

pub async fn update_course_details_db(
    pool: &PgPool,
    teacher_id: i32,
    id: i32,
    update_course: UpdateCourse,
) -> Result<Course, MyError> {
    let current_course = sqlx::query_as!(
        Course,
        "SELECT * FROM course where teacher_id = $1 and id = $2",
        teacher_id,
        id,
    )
    .fetch_one(pool)
    .await
    .map_err(|_err| MyError::NotFound("Course Id not found".into()))?;

    let name: String = if let Some(name) = update_course.name {
        name
    } else {
        current_course.name
    };

    let description: String = if let Some(desc) = update_course.description {
        desc
    } else {
        current_course.description.unwrap_or_default()
    };

    let format: String = if let Some(format) = update_course.format {
        format
    } else {
        current_course.format.unwrap_or_default()
    };

    let structure: String = if let Some(structure) = update_course.structure {
        structure
    } else {
        current_course.structure.unwrap_or_default()
    };

    let duration: String = if let Some(duration) = update_course.duration {
        duration
    } else {
        current_course.duration.unwrap_or_default()
    };

    let level: String = if let Some(level) = update_course.level {
        level
    } else {
        current_course.level.unwrap_or_default()
    };

    let language: String = if let Some(language) = update_course.language {
        language
    } else {
        current_course.language.unwrap_or_default()
    };

    let price: i32 = if let Some(price) = update_course.price {
        price
    } else {
        current_course.price.unwrap_or_default()
    };

    let course_row  = sqlx::query_as!(
        Course,
        "UPDATE course SET name = $1, description = $2, format = $3,
        structure = $4, duration = $5, price = $6, language = $7,
        level = $8 where teacher_id = $9 and id = $10
        RETURNING id, teacher_id, name, time,
        description, format, structure, duration, price, language, level",
        name,
        description,
        format,
        structure,
        duration,
        price,
        language,
        level,
        teacher_id,
        id,
    ).fetch_one(pool).await;

    if let Ok(course) = course_row {
        Ok(course)
    } else {
        Err(MyError::NotFound("Course id not found".into()))
    }

}
```

* src/models/mod.rs

```rust
pub mod course;
```

* src/models/course.rs

```rust
use crate::errors::MyError;
use actix_web::web;
use chrono::NaiveDateTime;
use serde::{Deserialize, Serialize}; // 序列化，反序列化

// use crate::models::course::Course

// Clone 用于解决所有权相关问题
// 使用 sqlx::FromRow 默认实现就可以将读表的结果自动映射成 Course struct
// 去掉 Deserialize 使其只用于查询，不需要从 json-> Course
#[derive(Serialize, Debug, Clone, sqlx::FromRow)]
pub struct Course {
    pub teacher_id: i32,
    pub id: i32,
    pub name: String,
    pub time: Option<NaiveDateTime>, // 时间类型

    // 新增字段
    pub description: Option<String>,
    pub format: Option<String>,
    pub structure: Option<String>,
    pub duration: Option<String>,
    pub price: Option<i32>,
    pub language: Option<String>,
    pub level: Option<String>,
}


// 新增时候用，只添加这些字段就行了
#[derive(Deserialize, Debug, Clone)]
pub struct CreateCourse {
    pub teacher_id: i32,
    pub name: String,
    pub description: Option<String>,
    pub format: Option<String>,
    pub structure: Option<String>,
    pub duration: Option<String>,
    pub price: Option<i32>,
    pub language: Option<String>,
    pub level: Option<String>,
}


// 实现 From 将 json 格式数据转为 Course
/*
    web::Json<T>、web::Data<T> 都属于叫数据提取器
    这里作用就是可以把 json 格式数据转为 Course 等特定类型的数据
*/
// impl From<web::Json<CreateCourse>> for CreateCourse {
//     fn from(course: web::Json<CreateCourse>) -> Self {
//         CreateCourse {
//             teacher_id: course.teacher_id,
//             name: course.name.clone(),
//             description: course.description.clone(),
//             format: course.format.clone(),
//             structure: course.structure.clone(),
//             duration: course.duration.clone(),
//             price: course.price,
//             language: course.language.clone(),
//             level: course.level.clone(),
//         }
//     } 
// }

// 怕出错可以实现 try_from trait，但实现 try_from trait 就必须注释掉 from trait
use std::convert::TryFrom;

impl TryFrom<web::Json<CreateCourse>> for CreateCourse {
    type Error = MyError;
    // Self 就是 CreateCourse
    fn try_from(course: web::Json<CreateCourse>) -> Result<Self, Self::Error> {
        Ok(CreateCourse {
                teacher_id: course.teacher_id,
                name: course.name.clone(),
                description: course.description.clone(),
                format: course.format.clone(),
                structure: course.structure.clone(),
                duration: course.duration.clone(),
                price: course.price,
                language: course.language.clone(),
                level: course.level.clone(),
            })
    }
}


// 用于修改课程，只能修改以下字段
#[derive(Debug, Deserialize, Clone)]
pub struct UpdateCourse {
    pub name: Option<String>,
    pub description: Option<String>,
    pub format: Option<String>,
    pub structure: Option<String>,
    pub duration: Option<String>,
    pub price: Option<i32>,
    pub language: Option<String>,
    pub level: Option<String>,
}

impl From<web::Json<UpdateCourse>> for UpdateCourse {
    fn from(course: web::Json<UpdateCourse>) -> Self {
        UpdateCourse { 
            name: course.name.clone(), 
            description: course.description.clone(), 
            format: course.format.clone(), 
            structure: course.structure.clone(), 
            duration: course.duration.clone(), 
            price: course.price, 
            language: course.language.clone(), 
            level: course.level.clone()
        }
    }
}
```

* src/handlers/mod.rs

```rust
pub mod course;
pub mod general;
```

* src/handlers/general.rs

```rust
// 处理通用的，这里只有健康检查

use crate::state::AppState;
use actix_web::{web, HttpResponse};

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
```

* src/handlers/course.rs

```rust
use crate::errors::MyError;
use crate::state::AppState;
use actix_web::{web, HttpResponse};
use crate::dbaccess::course::*;
use crate::models::course::{CreateCourse, UpdateCourse};
// use chrono::Utc;

// 经测试，app_state 与 new_course 顺序可互换
pub async fn post_new_course(
    app_state: web::Data<AppState>,
    new_course: web::Json<CreateCourse>, 
) -> Result<HttpResponse, MyError> {

    // new_course.into() 转为 Course 类型
    // CreateCourse 实现 try_from 所以这里应该改 into 为 try_into,
    // try_into()? 的 ? 表示转换时候可能出错
    post_new_course_db(&app_state.db, new_course.try_into()?)
    .await
    .map(|course| HttpResponse::Ok().json(course))
    
}

// GET localhost:3000/courses/{teacher_id}
// params: web::Path<(usize,)> 参数改为 web::Path(teacher_id): web::Path<i32> 等于提取了 teacher_id
pub async fn get_courses_for_teacher(
    app_state: web::Data<AppState>,
    //  web::Path<teacher_id: i32): web::Path<i32>,
    params: web::Path<i32>, // 这里应该是个元组，即 (usize,)
) -> Result<HttpResponse, MyError> {

    // 尝试转换 params.0 为 i32
    // let teacher_id = i32::try_from(params.0).unwrap();
    let teacher_id: i32 = params.into_inner();
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
    // web::Path((teacher_id, course_id)): web::Path<(i32, i32)>,
    params: web::Path<(i32, i32)>,
) -> Result<HttpResponse, MyError> {

    // let teacher_id = i32::try_from(params.0).unwrap();
    // let course_id = i32::try_from(params.1).unwrap();
    let (teacher_id, course_id) = params.into_inner();
    get_course_detail_db(&app_state.db, teacher_id, course_id)
    .await
    .map(|course| 
        HttpResponse::Ok().json(course)
    )
}


pub async fn delete_course(
    app_state: web::Data<AppState>,
    // web::Path((teacher_id, course_id)): web::Path<(i32, i32)>,
    params: web::Path<(i32, i32)>,
) -> Result<HttpResponse, MyError> {
    let (teacher_id, course_id) = params.into_inner();
    delete_course_db(&app_state.db, teacher_id, course_id)
    .await
    .map(|resp| HttpResponse::Ok().json(resp))
}

pub async fn update_course_details(
    app_state: web::Data<AppState>,
    update_course: web::Json<UpdateCourse>,
    params: web::Path<(i32, i32)>,
) -> Result<HttpResponse, MyError> {
    let (teacher_id, course_id) = params.into_inner();

    // UpdateCourse 实现了 from trait，所以这里调用 update_course.into()
    update_course_details_db(&app_state.db, teacher_id, course_id, update_course.into())
        .await
        .map(|course| HttpResponse::Ok().json(course))
}




#[cfg(test)]
mod tests {
    use super::*;
    use actix_web::{http::StatusCode, ResponseError};
    use chrono::Datelike;
    use std::sync::Mutex;
    // use chrono::NaiveDateTime;
    use dotenv::dotenv;
    use sqlx::postgres::PgPoolOptions;
    use std::env;

    // 用于忽略这个测试
    // #[ignore]
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
            CreateCourse {
                teacher_id: 1,
                name: "Brand New Course".into(), // 用 to_string() 也行
                description: Some("This is a course".into()),
                format: None,
                structure: None,
                duration: None,
                price: None,
                language: Some("en".into()),
                level: Some("Beginner".into()),
                // id: Some(4), // serial 类型，需要赋一个值
                // time: None,
            }
        );

        let resp = post_new_course(app_state, course).await.unwrap();
        println!("{:?}", resp);
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
        let teacher_id: web::Path<i32> = web::Path::from(1);
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
        let params: web::Path<(i32, i32)> = web::Path::from((1,6));
        let resp = get_course_detail(app_state, params).await.unwrap();
        assert_eq!(resp.status(), StatusCode::OK);
    }

    #[actix_rt::test]
    async fn get_one_course_failure() {
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
        let params: web::Path<(i32, i32)> = web::Path::from((1,100));
        let resp = get_course_detail(app_state, params).await;
        match resp {
            Ok(_) => println!("Something wrong ..."),
            Err(err) => assert_eq!(err.status_code(), StatusCode::NOT_FOUND),
        }
    }

    #[actix_rt::test]
    async fn update_course_success() {
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

        let update_course = UpdateCourse {
            name: Some("Course name changed".into()),
            description: Some("This is another test course".into()),
            format: None,
            level: Some("Intermediate".into()),
            price: None,
            duration: None,
            language: Some("Chinese".into()),
            structure: None,
        };

        let params: web::Path<(i32, i32)> = web::Path::from((1, 5));
        let update_param = web::Json(update_course);
        let resp = update_course_details(app_state, update_param, params).await.unwrap();
        assert_eq!(resp.status(), StatusCode::OK);
    }

    // #[ignore]
    #[actix_rt::test]
    async fn delete_course_success() {
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

        let params: web::Path<(i32, i32)> = web::Path::from((1, 7));
        let resp = delete_course(app_state, params).await.unwrap();
        assert_eq!(resp.status(), StatusCode::OK);
    } 

    #[actix_rt::test]
    async fn delete_course_failure() {
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

        let params: web::Path<(i32, i32)> = web::Path::from((1, 100));
        let resp = delete_course(app_state, params).await;
        match resp {
            Ok(_) => println!("Something wrong"),
            Err(err) => assert_eq!(err.status_code(), StatusCode::NOT_FOUND),
        }
    } 


}
```

* src/errors.rs

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
            // 未使用的变量前边加下划线 _ , 消除警告
            MyError::DBError(_msg) | MyError::ActixError(_msg) => StatusCode::INTERNAL_SERVER_ERROR,
            MyError::NotFound(_msg) => StatusCode::NOT_FOUND,
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

* src/routers.rs

```rust
use crate::handlers::{course::*, general::*};
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
    .route("/", web::post().to(post_new_course))
    // GET localhost:3000/courses/{teacher_id}
    .route("/{teacher_id}", web::get().to(get_courses_for_teacher))
    // GET localhost:3000/courses/{teacher_id}/{course_id}
    .route("/{teacher_id}/{course_id}", web::get().to(get_course_detail))
    .route("/{teacher_id}/{course_id}", web::delete().to(delete_course))
    .route("/{teacher_id}/{course_id}", web::put().to(update_course_details))
    );
}
```


* src/state.rs

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

* 命令

```shell
$ cargo test --bin teacher-service post_course_test
$ cargo test --bin teacher-service
$ cargo run
```

> 测试时要确保数据库数据是否正常
{: .prompt-info }

[重构代码](https://github.com/ZacharyWulven/Rust_Web_Full_Stack_Guide/commit/a9c76c794ca37f77f4b3af5710f69b1b2d05212d)
