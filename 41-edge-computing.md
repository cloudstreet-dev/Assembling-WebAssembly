# Chapter 41: Edge Computing with WebAssembly

## The Edge Computing Paradigm

Edge computing brings computation closer to users by running code at geographically distributed locations. WebAssembly is ideal for edge deployment:

- **Fast cold starts**: <1ms, crucial for edge workloads
- **Small binaries**: Rapid distribution across edge nodes
- **Portability**: Same code everywhere
- **Security**: Strong isolation for multi-tenant edge
- **Efficiency**: More instances per node

## Edge Platforms

### Cloudflare Workers

**Global network**: 300+ locations worldwide
**Execution**: V8 isolates with WASM support
**Cold start**: ~1ms
**Memory limit**: 128MB

**Complete edge application**:

```rust
use worker::*;
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct GeoRequest {
    lat: f64,
    lon: f64,
}

#[derive(Serialize)]
struct GeoResponse {
    country: String,
    city: String,
    distance_km: f64,
    edge_location: String,
}

#[event(fetch)]
pub async fn main(req: Request, env: Env, _ctx: Context) -> Result<Response> {
    let router = Router::new();

    router
        .get_async("/", |_, _| async move {
            Response::ok("Edge Computing with WASM")
        })
        .get_async("/geo", handle_geo)
        .post_async("/api/process", handle_process)
        .run(req, env)
        .await
}

async fn handle_geo(_req: Request, ctx: RouteContext<()>) -> Result<Response> {
    // Access edge metadata
    let cf = ctx.cf().unwrap();

    let response = GeoResponse {
        country: cf.country().unwrap_or("Unknown").to_string(),
        city: cf.city().unwrap_or("Unknown").to_string(),
        distance_km: 0.0,
        edge_location: cf.colo().unwrap_or("Unknown").to_string(),
    };

    Response::from_json(&response)
}

async fn handle_process(mut req: Request, ctx: RouteContext<()>) -> Result<Response> {
    let body = req.text().await?;

    // CPU-intensive processing in WASM
    let result = process_data(&body);

    Response::ok(result)
}

fn process_data(input: &str) -> String {
    // Example: Text processing
    input.to_uppercase()
}
```

**KV Storage** (edge data):

```rust
use worker::*;

#[event(fetch)]
pub async fn main(req: Request, env: Env, _ctx: Context) -> Result<Response> {
    let router = Router::new();

    router
        .get_async("/cache/:key", |_, ctx| async move {
            let key = ctx.param("key").unwrap();
            let kv = ctx.kv("MY_KV")?;

            match kv.get(key).text().await? {
                Some(value) => Response::ok(value),
                None => Response::error("Not found", 404),
            }
        })
        .post_async("/cache/:key", |mut req, ctx| async move {
            let key = ctx.param("key").unwrap();
            let value = req.text().await?;

            let kv = ctx.kv("MY_KV")?;
            kv.put(key, value)?
                .expiration_ttl(3600)  // 1 hour
                .execute()
                .await?;

            Response::ok("Cached")
        })
        .run(req, env)
        .await
}
```

**Durable Objects** (stateful edge):

```rust
use worker::*;

#[durable_object]
pub struct Counter {
    state: State,
    count: i32,
}

#[durable_object]
impl DurableObject for Counter {
    fn new(state: State, _env: Env) -> Self {
        Self { state, count: 0 }
    }

    async fn fetch(&mut self, _req: Request) -> Result<Response> {
        self.count += 1;

        // Persist state
        self.state
            .storage()
            .put("count", self.count)
            .await?;

        Response::ok(format!("Count: {}", self.count))
    }
}

#[event(fetch)]
pub async fn main(req: Request, env: Env, _ctx: Context) -> Result<Response> {
    let namespace = env.durable_object("COUNTER")?;
    let stub = namespace.id_from_name("my-counter")?.get_stub()?;

    stub.fetch_with_request(req).await
}
```

### Fastly Compute@Edge

**Network**: 70+ POPs globally
**Runtime**: Wasmtime (AOT compiled)
**Cold start**: <35ms

**Geolocation and routing**:

```rust
use fastly::http::StatusCode;
use fastly::{Error, Request, Response};

#[fastly::main]
fn main(req: Request) -> Result<Response, Error> {
    // Get client geolocation
    let geo = req.get_client_geo_info().unwrap();

    // Route based on location
    let backend = match geo.country_code() {
        "US" => "us_origin",
        "EU" => "eu_origin",
        "ASIA" => "asia_origin",
        _ => "global_origin",
    };

    // Forward to appropriate backend
    let backend_req = Request::get(format!("https://{}/api", backend))
        .with_header("X-Client-Country", geo.country_code());

    let mut resp = backend_req.send(backend)?;

    // Add edge headers
    resp.set_header("X-Edge-Location", geo.city());
    resp.set_header("X-Served-By", "Fastly-Compute");

    Ok(resp)
}
```

**Edge caching**:

```rust
use fastly::cache::{simple, Transaction};
use fastly::http::StatusCode;
use fastly::{Error, Request, Response};

#[fastly::main]
fn main(req: Request) -> Result<Response, Error> {
    let cache_key = format!("v1:{}", req.get_path());

    // Try cache
    match simple::get(&cache_key) {
        Ok(Some(mut entry)) => {
            // Cache hit
            let mut resp = Response::from_body(entry.to_stream());
            resp.set_header("X-Cache", "HIT");
            return Ok(resp);
        }
        Ok(None) => {
            // Cache miss, fetch from origin
        }
        Err(e) => {
            return Response::error(format!("Cache error: {}", e), 500);
        }
    }

    // Fetch from backend
    let backend_resp = req.clone_without_body().send("origin")?;

    // Cache response
    if backend_resp.get_status() == StatusCode::OK {
        let body = backend_resp.into_body_bytes();

        simple::set(&cache_key, body.clone(), 3600)?;  // 1 hour TTL

        let mut resp = Response::from_body(body);
        resp.set_header("X-Cache", "MISS");
        Ok(resp)
    } else {
        Ok(backend_resp)
    }
}
```

### Fermyon Spin (Edge)

**Deployment**: Fermyon Cloud edge locations
**Runtime**: Wasmtime

**Edge API with Redis**:

```rust
use spin_sdk::{
    http::{Request, Response},
    http_component,
    redis,
};
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct ViewRequest {
    post_id: String,
}

#[derive(Serialize)]
struct ViewResponse {
    post_id: String,
    views: i64,
}

#[http_component]
fn handle_request(req: Request) -> Result<Response> {
    let path = req.uri().path();

    match path {
        "/api/view" => handle_view(req),
        "/api/stats" => handle_stats(req),
        _ => Ok(Response::builder()
            .status(404)
            .body(Some("Not Found".into()))?),
    }
}

fn handle_view(req: Request) -> Result<Response> {
    let body = req.body().as_ref().unwrap();
    let view_req: ViewRequest = serde_json::from_slice(body)?;

    let key = format!("views:{}", view_req.post_id);

    // Increment view count in Redis
    let views = redis::incr(&key)?;

    // Set expiration (30 days)
    redis::execute("EXPIRE", &[&key, "2592000"])?;

    let response = ViewResponse {
        post_id: view_req.post_id,
        views,
    };

    Ok(Response::builder()
        .status(200)
        .header("Content-Type", "application/json")
        .body(Some(serde_json::to_string(&response)?.into()))?)
}

fn handle_stats(req: Request) -> Result<Response> {
    // Get all view counts
    let keys: Vec<String> = redis::execute("KEYS", &["views:*"])?
        .as_array()
        .unwrap()
        .iter()
        .map(|v| v.as_str().unwrap().to_string())
        .collect();

    let mut stats = vec![];

    for key in keys {
        let count: i64 = redis::get(&key)?.try_into()?;
        stats.push((key, count));
    }

    Ok(Response::builder()
        .status(200)
        .header("Content-Type", "application/json")
        .body(Some(serde_json::to_string(&stats)?.into()))?)
}
```

## Edge Computing Patterns

### Content Personalization

```rust
use fastly::{Request, Response, geo::Geo};

fn personalize_content(req: Request) -> Result<Response> {
    let geo = req.get_client_geo_info()?;
    let user_agent = req.get_header_str("User-Agent").unwrap_or("");

    // Device detection
    let device_type = if user_agent.contains("Mobile") {
        "mobile"
    } else {
        "desktop"
    };

    // Language detection
    let language = geo.country_code();

    // Fetch personalized content
    let content_url = format!(
        "https://origin/content?device={}&lang={}",
        device_type, language
    );

    let mut resp = Request::get(content_url).send("origin")?;

    // Add personalization headers
    resp.set_header("X-Device-Type", device_type);
    resp.set_header("X-Language", language);

    Ok(resp)
}
```

### A/B Testing at Edge

```rust
use sha2::{Sha256, Digest};

fn ab_test_assignment(user_id: &str, experiment: &str) -> &'static str {
    // Deterministic assignment based on hash
    let mut hasher = Sha256::new();
    hasher.update(format!("{}:{}", experiment, user_id));
    let result = hasher.finalize();

    // Use first byte for assignment
    let bucket = result[0] % 100;

    if bucket < 50 {
        "variant_a"
    } else {
        "variant_b"
    }
}

#[fastly::main]
fn main(req: Request) -> Result<Response> {
    let user_id = req.get_header_str("X-User-ID").unwrap_or("anonymous");

    let variant = ab_test_assignment(user_id, "homepage_redesign");

    // Route to appropriate backend
    let backend = match variant {
        "variant_a" => "origin_a",
        "variant_b" => "origin_b",
        _ => "origin_default",
    };

    let mut resp = req.clone_without_body().send(backend)?;

    resp.set_header("X-AB-Variant", variant);

    Ok(resp)
}
```

### Request Aggregation

```rust
use futures::future::join_all;

async fn aggregate_data(user_id: &str) -> Result<Response> {
    // Parallel requests to multiple services
    let profile_future = fetch_profile(user_id);
    let posts_future = fetch_posts(user_id);
    let stats_future = fetch_stats(user_id);

    // Wait for all
    let (profile, posts, stats) = tokio::join!(
        profile_future,
        posts_future,
        stats_future
    );

    // Combine responses
    let combined = serde_json::json!({
        "profile": profile?,
        "posts": posts?,
        "stats": stats?,
    });

    Ok(Response::from_json(&combined)?)
}

async fn fetch_profile(user_id: &str) -> Result<serde_json::Value> {
    let url = format!("https://api.example.com/users/{}", user_id);
    let resp = Request::get(url).send("api_backend").await?;
    Ok(resp.json().await?)
}

async fn fetch_posts(user_id: &str) -> Result<serde_json::Value> {
    let url = format!("https://api.example.com/posts?user={}", user_id);
    let resp = Request::get(url).send("api_backend").await?;
    Ok(resp.json().await?)
}

async fn fetch_stats(user_id: &str) -> Result<serde_json::Value> {
    let url = format!("https://api.example.com/stats/{}", user_id);
    let resp = Request::get(url).send("api_backend").await?;
    Ok(resp.json().await?)
}
```

### Edge Authentication

```rust
use jwt::VerifyWithKey;
use hmac::{Hmac, Mac};
use sha2::Sha256;

fn verify_jwt(token: &str, secret: &str) -> Result<bool> {
    let key: Hmac<Sha256> = Hmac::new_from_slice(secret.as_bytes())?;

    match token.verify_with_key(&key) {
        Ok(_) => Ok(true),
        Err(_) => Ok(false),
    }
}

#[fastly::main]
fn main(req: Request) -> Result<Response> {
    // Extract JWT from header
    let auth_header = req.get_header_str("Authorization").unwrap_or("");

    if !auth_header.starts_with("Bearer ") {
        return Response::error("Unauthorized", 401);
    }

    let token = &auth_header[7..];

    // Verify at edge
    let secret = std::env::var("JWT_SECRET").unwrap();

    if !verify_jwt(token, &secret)? {
        return Response::error("Invalid token", 401);
    }

    // Forward authenticated request
    req.send("origin")
}
```

## Performance Optimization

### Response Streaming

```rust
use fastly::http::body::StreamingBody;

#[fastly::main]
fn main(req: Request) -> Result<Response> {
    let mut backend_resp = req.send("origin")?;

    // Stream response body
    let mut resp = Response::builder()
        .status(backend_resp.get_status())
        .body(StreamingBody::new())?;

    // Copy headers
    for (name, value) in backend_resp.get_headers() {
        resp.set_header(name, value);
    }

    // Stream body chunks
    resp.stream_to_client();

    Ok(resp)
}
```

### Conditional Requests

```rust
fn handle_conditional(req: Request) -> Result<Response> {
    let cache_key = req.get_path();
    let if_none_match = req.get_header_str("If-None-Match");

    // Get cached ETag
    let cached_etag = get_cached_etag(&cache_key)?;

    if let (Some(client_etag), Some(cached)) = (if_none_match, cached_etag) {
        if client_etag == cached {
            // Client cache is valid
            return Ok(Response::builder()
                .status(304)  // Not Modified
                .header("ETag", cached)
                .body(None)?);
        }
    }

    // Fetch fresh content
    let mut resp = req.send("origin")?;

    // Cache new ETag
    if let Some(etag) = resp.get_header_str("ETag") {
        cache_etag(&cache_key, etag)?;
    }

    Ok(resp)
}
```

### Request Coalescing

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

struct CoalescingCache {
    pending: Arc<Mutex<HashMap<String, Arc<Mutex<Option<Response>>>>>>,
}

impl CoalescingCache {
    async fn get_or_fetch(&self, key: &str) -> Result<Response> {
        let pending_lock = Arc::clone(&self.pending);

        // Check if request already in flight
        let waiter = {
            let mut pending = pending_lock.lock().unwrap();

            if let Some(existing) = pending.get(key) {
                Arc::clone(existing)
            } else {
                let waiter = Arc::new(Mutex::new(None));
                pending.insert(key.to_string(), Arc::clone(&waiter));
                waiter
            }
        };

        // Try to get from waiter (if another request is fetching)
        {
            let response = waiter.lock().unwrap();
            if let Some(ref resp) = *response {
                return Ok(resp.clone());
            }
        }

        // Fetch (only one request will get here)
        let resp = fetch_from_origin(key).await?;

        // Store for other waiters
        {
            let mut response = waiter.lock().unwrap();
            *response = Some(resp.clone());
        }

        // Remove from pending
        {
            let mut pending = pending_lock.lock().unwrap();
            pending.remove(key);
        }

        Ok(resp)
    }
}
```

## Edge Storage Patterns

### Hierarchical Caching

```rust
async fn get_with_fallback(key: &str) -> Result<Option<String>> {
    // 1. Check edge cache (fast)
    if let Some(value) = edge_cache_get(key).await? {
        return Ok(Some(value));
    }

    // 2. Check regional cache
    if let Some(value) = regional_cache_get(key).await? {
        // Populate edge cache
        edge_cache_set(key, &value, 300).await?;  // 5 min TTL
        return Ok(Some(value));
    }

    // 3. Fetch from origin
    if let Some(value) = origin_fetch(key).await? {
        // Populate both caches
        edge_cache_set(key, &value, 300).await?;
        regional_cache_set(key, &value, 3600).await?;  // 1 hour TTL
        return Ok(Some(value));
    }

    Ok(None)
}
```

### Write-Through Cache

```rust
async fn update_with_cache_invalidation(key: &str, value: &str) -> Result<()> {
    // Update origin
    origin_update(key, value).await?;

    // Update edge cache
    edge_cache_set(key, value, 600).await?;

    // Invalidate related caches
    let related_keys = get_related_keys(key);
    for related in related_keys {
        edge_cache_delete(&related).await?;
    }

    Ok(())
}
```

## Monitoring and Debugging

### Edge Logging

```rust
use worker::console_log;

#[event(fetch)]
pub async fn main(req: Request, env: Env, ctx: Context) -> Result<Response> {
    let start = Date::now().as_millis();

    console_log!("Request: {} {}", req.method(), req.path());

    let result = handle_request(req, env, ctx).await;

    let duration = Date::now().as_millis() - start;

    console_log!("Duration: {}ms", duration);

    result
}
```

### Performance Metrics

```rust
use std::time::Instant;

struct Metrics {
    request_count: u64,
    total_duration_ms: u64,
    cache_hits: u64,
    cache_misses: u64,
}

impl Metrics {
    fn record_request(&mut self, duration_ms: u64, cache_hit: bool) {
        self.request_count += 1;
        self.total_duration_ms += duration_ms;

        if cache_hit {
            self.cache_hits += 1;
        } else {
            self.cache_misses += 1;
        }
    }

    fn report(&self) -> String {
        let avg_ms = if self.request_count > 0 {
            self.total_duration_ms / self.request_count
        } else {
            0
        };

        let hit_rate = if self.request_count > 0 {
            (self.cache_hits as f64 / self.request_count as f64) * 100.0
        } else {
            0.0
        };

        format!(
            "Requests: {}, Avg: {}ms, Cache hit rate: {:.1}%",
            self.request_count, avg_ms, hit_rate
        )
    }
}
```

## Security at the Edge

### Rate Limiting

```rust
async fn check_rate_limit(client_ip: &str) -> Result<bool> {
    let key = format!("ratelimit:{}", client_ip);

    // Get current count from edge KV
    let count: u32 = match kv_get(&key).await? {
        Some(n) => n.parse().unwrap_or(0),
        None => 0,
    };

    if count >= 100 {  // 100 requests per minute
        return Ok(false);
    }

    // Increment
    kv_set(&key, &(count + 1).to_string(), 60).await?;  // 60s TTL

    Ok(true)
}
```

### WAF Rules

```rust
fn check_waf_rules(req: &Request) -> Result<bool> {
    let path = req.get_path();
    let user_agent = req.get_header_str("User-Agent").unwrap_or("");

    // Block common attack patterns
    if path.contains("../") || path.contains("..\\") {
        return Ok(false);  // Path traversal
    }

    if path.contains("<script") || path.contains("javascript:") {
        return Ok(false);  // XSS attempt
    }

    // Block suspicious user agents
    if user_agent.contains("sqlmap") || user_agent.contains("nikto") {
        return Ok(false);  // Security scanner
    }

    Ok(true)
}
```

## Best Practices

1. **Minimize latency**: Choose nearest data source
2. **Cache aggressively**: Use multi-tier caching
3. **Stream responses**: Don't buffer large responses
4. **Handle failures gracefully**: Fallbacks for origin errors
5. **Monitor performance**: Track edge metrics
6. **Optimize binary size**: Smaller = faster distribution
7. **Use edge storage**: KV stores for configuration/data
8. **Test at scale**: Load testing from multiple regions
9. **Version deployments**: Gradual rollouts
10. **Secure by default**: Authentication, rate limiting, WAF

## Next Steps

Edge computing with WebAssembly brings computation closer to users, reducing latency and improving user experience. With platforms like Cloudflare Workers, Fastly Compute@Edge, and Fermyon Spin, you can deploy WASM applications globally with minimal overhead.

Next, we'll explore WebAssembly for IoT applications, where WASM enables secure, updatable firmware and extensible IoT devices.
