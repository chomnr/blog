+++
title = "technical breakdown: brute"
date = 2024-09-21
+++

A complete rewrite of my previous security monitoring tool, addressing major design flaws and performance issues. Brute monitors authentication attempts on your server with real-time processing and reliable data storage.

[GitHub Repository](https://github.com/zeljkovranjes/brute)

## What Was Wrong with BruteExpose

The original version had several fundamental problems:

**Geolocation Issues**
Used IPinfo's MMDB (MaxMind Database) instead of their API, causing errors with IP addresses not in the database. Required manual updates and because of that the data would get stale quick.

**JSON as Database**
Writing worked fine, but reading became problematic as file size grew. This impacted reliability regardless of server specs.

**Overcomplicated Data Structure**
Tracked password usage, usernames, metrics (hourly, daily, weekly), country data, and combinations through ObjectMapper. Created unnecessary complexity and maintenance headaches.

**Inefficient Log Processing**
The workflow was wasteful:

- Wait for OpenSSH to dump credentials to text file
- Read file contents
- Parse individual entries
- Parse entire log
- Store in JSON file

Required regular maintenance to prevent log files from growing too large. The only good part was the WebSocket implementation for real-time updates.

## How Brute Fixes Everything

Rebuilt from scratch in Rust for better memory safety, reliability, and performance.

**PostgreSQL Database**
Professional database implementation enables better data management, improved queries, and multi-server data collection.

**HTTP API Architecture**
Instead of monitoring text files, Brute runs an HTTP server with `/brute/attack/add` endpoint protected by bearer token authentication. Processes data immediately and broadcasts updates via WebSocket.

**Reliable IPInfo API**
Direct API integration eliminates missing IP issues and removes manual database updates.

**Actor-Based Design**
Clean architecture with specific handlers for each task, improving code organization and maintainability.

## Technical Implementation

Core dependencies handle web framework, database, and geolocation:

```toml
actix = "0.13.5"
actix-web = { version = "4", features = ["rustls-0_23"] }
sqlx = { version = "0.8.0", features = ["runtime-tokio", "tls-rustls", "postgres", "derive"] }
actix-web-actors = "4.3.0"
serde = { version = "1.0.130", features = ["derive"] }
serde_json = "1.0.122"
ipinfo = "3.0.0"
```

Initially used Axum but switched to Actix-web. The actor system implementation solved the original post-request handling issues.

The `/brute/attack/add` endpoint processes incoming attack data:

```rust
#[post("/attack/add")]
async fn post_brute_attack_add(
    state: web::Data<AppState>,
    payload: web::Json<AttackPayload>,
    bearer: BearerAuth,
) -> Result<HttpResponse, BruteResponeError> {
    // Verify bearer token
    if !bearer.token().eq(&state.bearer) {
        return Ok(HttpResponse::Unauthorized().body("body"));
    }

    // Reject local IP addresses
    if payload.ip_address.eq("127.0.0.1") {
        return Err(BruteResponeError::ValidationError("empty ip or local ip".to_string()));
    }

    // Create individual record
    let mut individual = Individual::new_short(
        payload.username.clone(),
        payload.password.clone(),
        payload.ip_address.clone(),
        payload.protocol.clone(),
    );

    // Validate data
    individual.validate()?;

    // Process data and broadcast results
    match state.actor.send(individual).await {
        Ok(res) => {
            websocket::BruteServer::broadcast(websocket::ParseType::ProcessedIndividual, res.unwrap());
            Ok(HttpResponse::Ok().into())
        },
        Err(er) => Err(BruteResponeError::InternalError(er.to_string())),
    }
}
```

The workflow is straightforward:

1. Verify authentication via bearer token
2. Validate IP address (no local addresses)
3. Perform data validation (string length, IP range verification)
4. Process through actor system
5. Broadcast results to WebSocket clients

The `BruteSystem` actor handles operations through dedicated handlers:

```rust
impl Handler<Individual> for BruteSystem {
    type Result = ResponseActFuture<Self, Result<ProcessedIndividual, BruteResponeError>>;

    fn handle(&mut self, msg: Individual, _: &mut Self::Context) -> Self::Result {
        let reporter = self.reporter();
        let fut = async move {
            match reporter.start_report(msg).await {
                Ok(result) => {
                    info!(
                        "Successfully processed Individual with ID: {}. Details: Username: '{}', IP: '{}', Protocol: '{}', Timestamp: {}, Location: {} - {}, {}, {}",
                        result.id(),
                        result.username(),
                        result.ip(),
                        result.protocol(),
                        result.timestamp(),
                        result.city().as_ref().unwrap_or(&"{EMPTY}".to_string()),
                        result.region().as_ref().unwrap_or(&"{EMPTY}".to_string()),
                        result.country().as_ref().unwrap_or(&"{EMPTY}".to_string()),
                        result.postal().as_ref().unwrap_or(&"{EMPTY}".to_string())
                    );
                    Ok(result)
                }
                Err(e) => {
                    error!("Failed to process report: {}", e);
                    Err(BruteResponeError::InternalError(
                        "something definitely broke on our side".to_string(),
                    ))
                }
            }
        };
        fut.into_actor(self).map(|res, _, _| res).boxed_local()
    }
}
```

The `reporter.start_report()` method initiates database transactions for storing and processing attack data.

Database interactions use a standardized `Reportable` trait:

```rust
#[allow(async_fn_in_trait)]
pub trait Reportable<T, R> {
      async fn report<'a>(reporter: &T, model: &'a R) -> anyhow::Result<R>
      where
          Self: Sized;
}
```

Implementation for the `Individual` model:

```rust
impl Reportable<BruteReporter, Individual> for Individual {
    async fn report<'a>(
        reporter: &BruteReporter,
        model: &'a Individual,
    ) -> anyhow::Result<Individual> {
        let pool = &reporter.brute.db_pool;
        let query = r#"
            INSERT INTO individual (id, username, password, ip, protocol, timestamp)
            VALUES ($1, $2, $3, $4, $5, $6)
            RETURNING *
        "#;

        // Generate new ID and timestamp for the new instance
        let new_id = Uuid::new_v4().as_simple().to_string();
        let new_timestamp = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_millis() as i64;

        // Execute the query and get the inserted data
        let inserted = sqlx::query_as::<_, Individual>(query)
            .bind(&new_id)
            .bind(&model.username())
            .bind(&model.password())
            .bind(&model.ip())
            .bind(&model.protocol())
            .bind(new_timestamp)
            .fetch_one(pool)
            .await?;

        Ok(inserted)
    }
}
```

This creates database entries with clean, maintainable patterns.

To minimize IPInfo API usage, a caching system stores IP data:

1. Check if IP exists in cache
2. If found and less than 5 minutes old, use cached data
3. If not found or stale, fetch from API
4. Store results in cache

This reduces API requests while maintaining data accuracy.

The rewrite is significantly better than the original. Running in a Docker container for three days with zero errors proves the architecture is solid and well-designed.
