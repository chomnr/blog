+++
title = "Technical Breakdown: Brute"
date = 2024-09-21
+++

# Introducing Brute: Monitors authentication attempts on your server

I'm excited to announce the release of my latest project, **Brute**. This application represents a complete redevelopment of my previous attempt at a security monitoring tool, addressing numerous design flaws and performance issues.

For those unfamiliar with the project, I encourage you to check out the [GitHub repository](https://github.com/chomnr/brute) for full details and implementation.

## The Predecessor: BruteExpose

Now you might be wondering what was wrong with its predecessor BruteExpose, well, everything. Here's a list:

### Key Issues

1. **Inefficient Geolocation Implementation**  
   I relied on IPinfo's MMDB (MaxMind Database) instead of their API, which caused errors when processing IP addresses not found in the database. The database required manual updates, creating maintenance overhead and reliability issues.

2. **Poor Data Storage Strategy**  
   Using JSON as a database proved to be a significant mistake. While writing operations functioned adequately, reading operations became problematic once the file reached a certain size. This limitation seriously impacted reliability, regardless of server specifications.

3. **Overly Complex Data Structure**  
   The JSON file structure—managed through an ObjectMapper—was highly organized but unnecessarily complex. It tracked password usage, usernames, various metrics (hourly, daily, weekly), country data, and username/password combinations. This approach cluttered the codebase and created maintenance challenges.

4. **Inefficient Log Processing**  
   BruteExpose relied on monitoring a text file for SSH login attempts. The application would:

   - Wait for OpenSSH to dump credentials to a text file
   - Read the file contents
   - Parse individual log entries
   - Parse the entire log
   - Store results in the JSON file

   This approach was inefficient and required regular maintenance to prevent the log file from growing too large.

The only positive aspect of BruteExpose was its WebSocket implementation, which allowed for real-time updates.

## The Successor: Brute

For the redevelopment, I chose Rust instead of Java for its reliability, memory safety, and performance benefits. Here's how Brute improves upon its predecessor:

### Major Improvements

1. **Language Choice**  
   Rust provides superior memory safety and reliability compared to Java, making it ideal for a security-focused application.

2. **Professional Database Implementation**  
   I replaced the JSON-based storage with PostgreSQL, enabling better data management, improved query capabilities, and the ability to collect data from multiple "farmer" servers.

3. **Modern API Architecture**  
   Instead of monitoring text files for changes, Brute implements an HTTP server with dedicated endpoints (/brute/attack/add) protected by bearer token authentication. This approach processes data immediately and broadcasts updates to WebSocket clients in real-time.

4. **Reliable Geolocation**  
   Implementing the IPInfo API eliminated issues with missing IP addresses and removed the need for manual database updates.

5. **Actor-Based Architecture**  
   Without the constraints of a JSON database, I adopted an actor-based design pattern, assigning specific handlers for each task to improve code organization and maintainability.

## Technical Implementation

### Core Dependencies

```toml
# Main crates
actix = "0.13.5"
actix-web = { version = "4", features = ["rustls-0_23"] }
sqlx = { version = "0.8.0", features = ["runtime-tokio", "tls-rustls", "postgres", "derive"] }
actix-web-actors = "4.3.0"
serde = { version = "1.0.130", features = ["derive"] }
serde_json = "1.0.122"
ipinfo = "3.0.0"
```

I initially used Axum as the web framework but switched to Actix-web due to specific issues. The transition didn't immediately resolve my problem, and the ultimate fix likely involved implementing the actor system to handle post-request mechanisms correctly.

### HTTP Endpoint Implementation

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

The endpoint follows a clear workflow:

1. Verify authentication via bearer token
2. Validate IP address (no local addresses)
3. Perform comprehensive validation (string length, IP range verification)
4. Process data through the actor system
5. Broadcast results to WebSocket clients

### Actor System

The `BruteSystem` actor handles various operations through dedicated handlers:

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

The `reporter.start_report()` method initiates a transaction for database operations, managing the process of storing and processing attack data.

### Reporter System

The application uses a `Reportable` trait to standardize database interactions:

```rust
#[allow(async_fn_in_trait)]
pub trait Reportable<T, R> {
      async fn report<'a>(reporter: &T, model: &'a R) -> anyhow::Result<R>
      where
          Self: Sized;
}
```

Implementation example for the `Individual` model:

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

This creates a database entry with the specified values in a clean, maintainable pattern.

### Optimized IPInfo Integration

To minimize API usage, I implemented a caching system for IP data:

1. Check if the IP exists in the cache
2. If found and entry is less than 5 minutes old, use cached data
3. If not found or data is stale, fetch from the API
4. Store results in cache

This approach significantly reduces API requests while maintaining data accuracy.

## Conclusion

The successor is 100000% better than the predecessor, as it should be; the project has been running inside of a docker container for about three days now and has had no errors! So, I can safely say this project was well thought out
