# Backend Foundations

These core foundation modules establish the response envelope, error mapping, configuration parsing, database pooling, dependency injection, and server initialization for Actix-web backends.

> [!TIP]
> Ready-to-use boilerplate templates for these foundation primitives are located in [`templates/backend/src/`](../../templates/backend/src/).

---

## 1. Unified API Response Envelope

Every endpoint returns a standardized JSON structure with payload, status code, optional human message, and UTC timestamp.

### `entities/base_response.rs`
```rust
use actix_web::web::Json;
use chrono::{DateTime, Utc};
use serde::Serialize;

use crate::entities::app_response::AppResponse;

#[derive(Debug, Serialize)]
pub struct BaseResponse<T: Serialize> {
    pub data: Option<T>,
    pub status: u16,
    pub message: Option<String>,
    pub timestamp: DateTime<Utc>,
}

impl<T: Serialize> BaseResponse<T> {
    pub fn success(data: T) -> Self {
        Self {
            data: Some(data),
            status: 200,
            message: None,
            timestamp: Utc::now(),
        }
    }
}

impl BaseResponse<()> {
    pub fn message(message: impl Into<String>) -> Self {
        Self {
            data: None,
            status: 200,
            message: Some(message.into()),
            timestamp: Utc::now(),
        }
    }

    pub fn error(status: u16, message: impl Into<String>) -> Self {
        Self {
            data: None,
            status,
            message: Some(message.into()),
            timestamp: Utc::now(),
        }
    }
}

pub trait JsonFromStringTrait {
    fn json(self) -> AppResponse<String>;
    fn json_data(self) -> AppResponse<String>;
}

impl JsonFromStringTrait for String {
    fn json(self) -> AppResponse<String> {
        Ok(Json(BaseResponse {
            data: None,
            status: 200,
            message: Some(self),
            timestamp: Utc::now(),
        }))
    }

    fn json_data(self) -> AppResponse<String> {
        Ok(Json(BaseResponse::success(self)))
    }
}
```

### `entities/app_response.rs`
> [!TIP]
> Conversion traits allow handlers to return `AppResponse<T>` cleanly:
> - Use `.json()` on a `Result<T, AppError>` (e.g., `service.get_user().await.json()`)
> - Use `String.json_data()` on a direct payload `T` (e.g., `String::from("ok").json_data()`)
> - Use `String.json()` to return a success message without data (e.g., `String::from("Logged out").json()`)

```rust
use actix_web::web::Json;
use serde::Serialize;

use crate::entities::{app_error::AppError, base_response::BaseResponse};

pub type AppResponse<T> = Result<Json<BaseResponse<T>>, AppError>;

pub trait IntoResponseTrait<T: Serialize> {
    fn json(self) -> AppResponse<T>;
}

impl<T: Serialize> IntoResponseTrait<T> for Result<T, AppError> {
    fn json(self) -> AppResponse<T> {
        self.map(BaseResponse::success).map(Json)
    }
}
```

---

## 2. Centralized Error Handling

Map domain errors to HTTP status codes while keeping internal errors opaque to clients for security.

### `entities/app_error.rs`
```rust
use actix_web::{http::StatusCode, HttpResponse, ResponseError};
use thiserror::Error;

use crate::entities::base_response::BaseResponse;

#[derive(Debug, Error)]
pub enum AppError {
    #[error("{0}")]
    BadRequest(String),

    #[error("{0}")]
    Unauthorized(String),

    #[error("{0}")]
    Forbidden(String),

    #[error("{0}")]
    NotFound(String),

    #[error("{0}")]
    Conflict(String),

    #[error("an internal server error occurred")]
    Internal,
}

impl AppError {
    pub fn bad_request(message: impl Into<String>) -> Self {
        Self::BadRequest(message.into())
    }

    pub fn unauthorized(message: impl Into<String>) -> Self {
        Self::Unauthorized(message.into())
    }

    pub fn not_found(message: impl Into<String>) -> Self {
        Self::NotFound(message.into())
    }

    pub fn conflict(message: impl Into<String>) -> Self {
        Self::Conflict(message.into())
    }
}

impl ResponseError for AppError {
    fn status_code(&self) -> StatusCode {
        match self {
            Self::BadRequest(_) => StatusCode::BAD_REQUEST,
            Self::Unauthorized(_) => StatusCode::UNAUTHORIZED,
            Self::Forbidden(_) => StatusCode::FORBIDDEN,
            Self::NotFound(_) => StatusCode::NOT_FOUND,
            Self::Conflict(_) => StatusCode::CONFLICT,
            Self::Internal => StatusCode::INTERNAL_SERVER_ERROR,
        }
    }

    fn error_response(&self) -> HttpResponse {
        let status = self.status_code();
        let message = match self {
            Self::Internal => "Internal server error".to_owned(),
            _ => self.to_string(),
        };
        HttpResponse::build(status).json(BaseResponse::<()>::error(status.as_u16(), message))
    }
}

impl From<sqlx::Error> for AppError {
    fn from(error: sqlx::Error) -> Self {
        eprintln!("database error: {error}");
        Self::Internal
    }
}

impl From<redis::RedisError> for AppError {
    fn from(error: redis::RedisError) -> Self {
        eprintln!("redis error: {error}");
        Self::Internal
    }
}

impl From<jsonwebtoken::errors::Error> for AppError {
    fn from(error: jsonwebtoken::errors::Error) -> Self {
        eprintln!("jwt error: {error}");
        Self::Internal
    }
}
```

---

## 3. Strongly Typed Configuration

Read and validate all required environment variables once during application startup.

### `entities/app_config.rs`
```rust
use std::env;

#[derive(Clone, Debug)]
pub struct AppConfig {
    pub app_name: String,
    pub bind_address: String,
    pub port: u16,
    pub database_url: String,
    pub redis_url: String,
    pub jwt_secret: String,
    pub access_token_expiration_seconds: u64,
    pub refresh_token_expiration_seconds: u64,
    pub cors_allowed_origin: String,
}

impl AppConfig {
    pub fn from_env() -> Result<Self, String> {
        Ok(Self {
            app_name: required("APP_NAME")?,
            bind_address: env::var("BIND_ADDRESS").unwrap_or_else(|_| "127.0.0.1".into()),
            port: required("PORT")?
                .parse()
                .map_err(|_| "PORT must be an integer")?,
            database_url: required("DATABASE_URL")?,
            redis_url: env::var("REDIS_URL").unwrap_or_else(|_| "redis://127.0.0.1:6379".into()),
            jwt_secret: required("JWT_SECRET")?,
            access_token_expiration_seconds: required("ACCESS_TOKEN_EXPIRATION_SECONDS")?
                .parse()
                .map_err(|_| "ACCESS_TOKEN_EXPIRATION_SECONDS must be an integer")?,
            refresh_token_expiration_seconds: required("REFRESH_TOKEN_EXPIRATION_SECONDS")?
                .parse()
                .map_err(|_| "REFRESH_TOKEN_EXPIRATION_SECONDS must be an integer")?,
            cors_allowed_origin: env::var("CORS_ALLOWED_ORIGIN")
                .unwrap_or_else(|_| "http://localhost:3000".into()),
        })
    }
}

fn required(name: &str) -> Result<String, String> {
    env::var(name).map_err(|_| format!("missing required environment variable: {name}"))
}
```

---

## 4. Infrastructure Setup Modules

### `setup/setup_db.rs`: PostgreSQL Pool & Migrations
Connect to the database pool and run pending compile-time migrations upon startup:

```rust
use sqlx::{postgres::PgPoolOptions, PgPool};
use crate::entities::{app_config::AppConfig, app_error::AppError};

pub async fn setup_db(config: &AppConfig) -> Result<PgPool, AppError> {
    let pool = PgPoolOptions::new()
        .max_connections(10)
        .connect(&config.database_url)
        .await?;

    sqlx::migrate!().run(&pool).await.map_err(|error| {
        eprintln!("database migration error: {error}");
        AppError::Internal
    })?;

    Ok(pool)
}
```

### `setup/setup_http_client.rs`: Outbound HTTP Client
Initialize a shared Reqwest client configured with connection timeouts:

```rust
use reqwest::Client;
use std::time::Duration;
use crate::entities::app_error::AppError;

pub fn setup_http_client() -> Result<Client, AppError> {
    Client::builder()
        .timeout(Duration::from_secs(30))
        .build()
        .map_err(|error| {
            eprintln!("HTTP client initialization error: {error}");
            AppError::Internal
        })
}
```

---

## 5. Dependency Injection Container

Construct all pools, clients, repositories, and services once, then inject them into Actix web application state as `actix_web::web::Data<T>`.

### `di.rs`
```rust
use actix_web::web::{self, Data, ServiceConfig};
use reqwest::Client;
use sqlx::PgPool;

use crate::{
    entities::{app_config::AppConfig, app_error::AppError},
    setup::{setup_db::setup_db, setup_http_client::setup_http_client},
};

#[derive(Clone)]
pub struct AppDependencies {
    pub config: Data<AppConfig>,
    pub db: Data<PgPool>,
    pub http_client: Data<Client>,
    // Add additional repositories and services here
}

impl AppDependencies {
    pub async fn build(config: AppConfig) -> Result<Self, AppError> {
        let db = setup_db(&config).await?;
        let http_client = setup_http_client()?;

        Ok(Self {
            config: Data::new(config),
            db: Data::new(db),
            http_client: Data::new(http_client),
        })
    }

    pub fn configure(&self, cfg: &mut ServiceConfig) {
        cfg.app_data(self.config.clone())
            .app_data(self.db.clone())
            .app_data(self.http_client.clone());
    }
}
```

---

## 6. HTTP Server Wiring & Central Route Configuration

### `http.rs`: JSON Parsing Error Handling & CORS
Support multi-origin parsing, localhost/127.0.0.1 mapping, credentials, and standardized JSON error formatting:

```rust
use actix_cors::Cors;
use actix_web::{error::InternalError, web, ResponseError};

use crate::entities::app_error::AppError;

pub fn cors(allowed_origin: &str) -> Cors {
    let mut cors = Cors::default()
        .allow_any_method()
        .allow_any_header()
        .supports_credentials()
        .max_age(3600);

    for origin in allowed_origin.split(',') {
        let trimmed = origin.trim();
        if !trimmed.is_empty() {
            cors = cors.allowed_origin(trimmed);
        }
    }

    if allowed_origin.contains("localhost:3000") && !allowed_origin.contains("127.0.0.1:3000") {
        cors = cors.allowed_origin("http://127.0.0.1:3000");
    } else if allowed_origin.contains("127.0.0.1:3000")
        && !allowed_origin.contains("localhost:3000")
    {
        cors = cors.allowed_origin("http://localhost:3000");
    }

    cors
}

pub fn json_config() -> web::JsonConfig {
    web::JsonConfig::default().error_handler(|_, _| {
        InternalError::from_response(
            "invalid JSON body",
            AppError::bad_request("Invalid JSON request body").error_response(),
        )
        .into()
    })
}
```

### `main.rs`: Runtime Bootstrap
Load local `.env` via `dotenvy`, build dependency container, and start HTTP server:

```rust
use std::io;

use actix_web::{middleware, App, HttpServer};
use dotenvy::{dotenv, Error as DotenvError};

use backend::{di::AppDependencies, entities::app_config::AppConfig, http, routes};

#[actix_web::main]
async fn main() -> io::Result<()> {
    if let Err(error) = dotenv() {
        if !matches!(error, DotenvError::Io(ref source) if source.kind() == io::ErrorKind::NotFound)
        {
            return Err(io::Error::new(
                io::ErrorKind::InvalidData,
                format!("failed to load .env: {error}"),
            ));
        }
    }

    let config = AppConfig::from_env()
        .map_err(|error| io::Error::new(io::ErrorKind::InvalidInput, error))?;
    let dependencies = AppDependencies::build(config.clone())
        .await
        .map_err(|error| io::Error::new(io::ErrorKind::ConnectionRefused, error.to_string()))?;
    let bind_address = config.bind_address.clone();
    let port = config.port;

    println!("Launching {} on http://{bind_address}:{port}", config.app_name);

    HttpServer::new(move || {
        App::new()
            .wrap(middleware::Compress::default())
            .wrap(middleware::NormalizePath::trim())
            .wrap(http::cors(&config.cors_allowed_origin))
            .app_data(http::json_config())
            .configure(|service_config| dependencies.configure(service_config))
            .configure(routes::configure)
            .default_service(actix_web::web::route().to(routes::not_found))
    })
    .bind((bind_address, port))?
    .run()
    .await
}
```

### `routes/mod.rs`: Index, Health, & 404 Handlers
```rust
use actix_web::{get, web, HttpResponse};
use crate::entities::app_response::AppResponse;
use crate::entities::base_response::{BaseResponse, JsonFromStringTrait};

pub fn configure(cfg: &mut web::ServiceConfig) {
    cfg.service(index)
       .service(health);
}

#[get("/")]
pub async fn index() -> AppResponse<String> {
    String::from("API service running").json_data()
}

#[get("/health")]
pub async fn health() -> AppResponse<String> {
    String::from("ok").json_data()
}

pub async fn not_found() -> HttpResponse {
    HttpResponse::NotFound().json(BaseResponse::<()>::error(404, "Not found"))
}
```
