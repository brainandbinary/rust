In Axum, an **Extractor** is a mechanism for pulling data out of an incoming HTTP request (like headers, query parameters, path variables, form payloads, or cookies) and turning it into strongly-typed Rust values inside your route handler arguments.

Instead of manually parsing headers, reading body streams, or deserializing JSON, you place an Extractor as a parameter in your function signature, and Axum handles all the parsing, validation, and error management automatically before your handler function ever runs.

---

## 1. How Extractors Work

When a request hits a route, Axum checks the function signature of your handler, runs each extractor in order, and only invokes your function if **all extractors succeed**.

```rust
pub async fn login_handler(
    token: CsrfToken,                     // Extracted from Headers/Cookies
    State(app_state): State<AppState>,     // Extracted from Application State
    Form(payload): Form<LoginForm>,       // Extracted from HTTP Request Body
) -> Result<Response, ...> {
    // If execution reaches here, ALL three extractors succeeded!
}

```

If any extractor fails (for example, `Form` fails because the user sent malformed data), Axum stops execution and returns an HTTP error response (like `400 Bad Request` or `422 Unprocessable Entity`) automatically.

---

## 2. Common Built-in Extractors

Axum provides many built-in extractors for common web development tasks:

| Extractor | What it extracts | Example Usage |
| --- | --- | --- |
| **`Path<T>`** | Dynamic URI parameters | `/users/:id` $\rightarrow$ `Path(user_id): Path<u32>` |
| **`Query<T>`** | Query string parameters | `/search?q=rust` $\rightarrow$ `Query(params): Query<SearchParams>` |
| **`Form<T>`** | URL-encoded HTML form data | `<form method="post">` $\rightarrow$ `Form(payload): Form<LoginForm>` |
| **`Json<T>`** | JSON body payload | `{"key": "val"}` $\rightarrow$ `Json(data): Json<MyStruct>` |
| **`State<T>`** | Shared app state | Shared database pool or configuration |
| **`HeaderMap`** | Raw HTTP headers | Reading `User-Agent`, `Authorization`, etc. |
| **`CookieJar`** | Cookies from request headers | `jar.get("session")` (via `axum-extra`) |

---

## 3. How Axum Implements Extractors Under the Hood

Under the hood, an Extractor is simply any type that implements one of two Axum traits:

### 1. `FromRequestParts`

Extracts data **only from the metadata** of the request (URI, headers, cookies, extensions).

* **Key characteristic:** Multiple `FromRequestParts` extractors can run on a single request without interfering with each other.
* *Examples:* `Path`, `Query`, `HeaderMap`, `CsrfToken`, `State`.

### 2. `FromRequest`

Extracts data from the **HTTP request body**.

* **Key characteristic:** The HTTP request body is a streaming payload that can only be read/consumed **once**.
* *Examples:* `Form`, `Json`, `String`, `Bytes`.

> **The Golden Rule of Axum Extractors:** Because `FromRequest` consumes the body stream, **a body extractor MUST be the very last parameter** in your function signature.

---

## 4. Creating Your Own Custom Extractor

You can build custom extractors by implementing `FromRequestParts`. This is the standard way to handle tasks like **authentication** or **permission checking** cleanly across multiple routes:

```rust
use axum::{
    async_trait,
    extract::FromRequestParts,
    http::{request::Parts, StatusCode},
};

// Custom type representing an authenticated user
pub struct AuthUser {
    pub user_id: String,
}

#[async_trait]
impl<S> FromRequestParts<S> for AuthUser
where
    S: Send + Sync,
{
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        // Extract Authorization header
        if let Some(auth_header) = parts.headers.get("Authorization") {
            if let Ok(header_str) = auth_header.to_str() {
                if header_str.starts_with("Bearer ") {
                    return Ok(AuthUser { user_id: "123".to_string() });
                }
            }
        }

        // Reject if missing or invalid
        Err((StatusCode::UNAUTHORIZED, "Missing or invalid token"))
    }
}

// Handler using the custom extractor
pub async fn protected_dashboard(user: AuthUser) -> String {
    format!("Welcome back, user {}!", user.user_id)
}

```

---

## Summary

Extractors are Axum's way of achieving **declarative type safety**. Rather than writing boilerplate parsing code inside your route functions, you specify *what data your handler needs* in its parameters, and let Axum parse, validate, and inject it safely.
