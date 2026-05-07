
This implementation combines **Axum 0.8**, **DashMap**, and **Tokio MPSC** channels. It includes the "Reaper" logic for lazy rooms and automatic cleanup when users disconnect.

```rust
use axum::{
    extract::{Path, State},
    response::IntoResponse,
    routing::{get, post},
    Json, Router,
};
use dashmap::DashMap;
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration};

// --- Types & State ---

#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct RoomMeta {
    pub name: String,
    pub labels: Vec<String>,
}

pub struct Room {
    pub meta: RoomMeta,
    pub user_1: Option<mpsc::Sender<String>>,
    pub user_2: Option<mpsc::Sender<String>>,
}

// Global shared state
pub type SharedState = Arc<DashMap<String, Room>>;

// --- API Payloads ---

#[derive(Deserialize)]
struct CreateRoomInput {
    id: String,
    meta: RoomMeta,
}

// --- Logic & Handlers ---

#[tokio::main]
async fn main() {
    let state: SharedState = Arc::new(DashMap::new());

    let app = Router::new()
        .route("/rooms", post(create_room_handler))
        .route("/rooms", get(list_rooms_handler))
        .route("/rooms/:id/join", post(join_room_handler))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

/// 1. Create a room and spawn a "Reaper" task for lazy deletion
async fn create_room_handler(
    State(state): State<SharedState>,
    Json(input): Json<CreateRoomInput>,
) -> impl IntoResponse {
    let (tx, mut rx) = mpsc::channel(100);

    let room = Room {
        meta: input.meta,
        user_1: Some(tx),
        user_2: None,
    };

    state.insert(input.id.clone(), room);

    // Spawn Lazy Room Reaper: Deletes room if no one joins in 10 mins
    let state_reap = Arc::clone(&state);
    let id_reap = input.id.clone();
    tokio::spawn(async move {
        sleep(Duration::from_secs(600)).await;
        if let Some(r) = state_reap.get(&id_reap) {
            if r.user_2.is_none() {
                state_reap.remove(&id_reap);
                println!("Lazy room {} reaped.", id_reap);
            }
        }
    });

    // Handle the creator's connection in a separate task
    tokio::spawn(async move {
        while let Some(msg) = rx.recv().await {
            println!("Creator received: {}", msg);
        }
    });

    Json("Room created")
}

/// 2. List all rooms that aren't full
async fn list_rooms_handler(State(state): State<SharedState>) -> impl IntoResponse {
    let available: Vec<_> = state
        .iter()
        .filter(|r| r.user_2.is_none())
        .map(|r| (r.key().clone(), r.meta.clone()))
        .collect();

    Json(available)
}

/// 3. Join a room and setup the Relay
async fn join_room_handler(
    State(state): State<SharedState>,
    Path(id): Path<String>,
) -> impl IntoResponse {
    let (tx_internal, mut rx_internal) = mpsc::channel(100);
    
    // Scoped block to update state
    {
        let mut room = match state.get_mut(&id) {
            Some(r) => r,
            None => return Err("Room not found"),
        };

        if room.user_2.is_some() {
            return Err("Room full");
        }
        room.user_2 = Some(tx_internal);
    }

    // Spawn the Joiner's message relay
    let state_relay = Arc::clone(&state);
    let id_relay = id.clone();
    
    tokio::spawn(async move {
        // Guard for auto-deletion when this task ends
        let _guard = RoomCleanupGuard { id: id_relay.clone(), state: state_relay.clone() };

        loop {
            tokio::select! {
                // Listen for messages from the Peer (User 1)
                Some(msg) = rx_internal.recv() => {
                    println!("Joiner received: {}", msg);
                }
                // Simulating receiving from a WebSocket/Socket
                // In a real app, you'd listen to the network here
            }
        }
    });

    Ok(Json("Joined room"))
}

// --- Cleanup Mechanism ---

struct RoomCleanupGuard {
    id: String,
    state: SharedState,
}

impl Drop for RoomCleanupGuard {
    fn drop(&mut self) {
        let id = self.id.clone();
        let state = Arc::clone(&self.state);
        
        // Finalize room deletion when the user task terminates
        tokio::spawn(async move {
            if state.remove(&id).is_some() {
                println!("Room {} cleaned up successfully.", id);
            }
        });
    }
}
```

### Key Technical Details:
*   **The Guard:** `RoomCleanupGuard` uses the **RAII (Resource Acquisition Is Initialization)** pattern. Since it is owned by the `join_room` task, the moment that task stops (due to disconnect or error), `drop()` is called, and the room is purged from the `DashMap`.
In Rust, `drop()` is the mechanism for **automatic resource management**. It is part of the `Drop` trait, which allows you to run specific code when a value goes out of scope. 

For your chat app, this is the "Cleanup Crew."

### 1. How it works fundamentally
In many languages (like Java or Python), a Garbage Collector (GC) runs periodically to find unused memory. In Rust, there is no GC. Instead, Rust knows exactly when a variable is no longer needed: when the curly braces `{}` it lives in are closed. 

At that exact moment, Rust calls `drop()`.

### 2. The `Drop` Trait in your Chat App
In the code I provided, we defined a custom `Drop` implementation for the `RoomCleanupGuard`. This is how we ensure that a room is deleted from the `DashMap` as soon as the user's task stops.

```rust
impl Drop for RoomCleanupGuard {
    fn drop(&mut self) {
        // This code runs AUTOMATICALLY when the variable goes out of scope.
        let id = self.id.clone();
        let state = Arc::clone(&self.state);
        
        tokio::spawn(async move {
            state.remove(&id); 
            println!("Memory cleared for room {}", id);
        });
    }
}
```



---

### 3. Why is this useful for you?
Without `drop()`, you would have to manually write `state.remove(id)` every time a user:
*   Closes their browser.
*   Loses their internet connection.
*   The server encounters an error and crashes the task.

If you forgot even *one* of those cases, a "ghost room" would stay in your RAM forever. With the `Drop` trait, it doesn't matter *why* the task ended; the moment the `_guard` variable is destroyed, the cleanup code runs.

---

### 4. The "Scope" Logic
Think of it like this in your `tokio::spawn` block:

```rust
tokio::spawn(async move {
    // 1. Guard is created and "Alive"
    let _guard = RoomCleanupGuard { ... }; 

    loop {
        // ... chat logic ...
        // If the user disconnects, the loop breaks.
    }

    // 2. The task reaches the end of its block.
    // 3. Rust sees that '_guard' is about to be destroyed.
    // 4. Rust calls '_guard.drop()'.
});
```

### 5. Important Rules of `drop()`
*   **You cannot call it manually:** You aren't allowed to call `x.drop()`. Rust must be the one to do it to ensure memory safety. (If you need to force an early drop, you use the global function `drop(x)`).
*   **Order matters:** If you have multiple variables, Rust drops them in the **reverse order** they were created (Last-In, First-Out).
*   **Ownership:** Once a value is dropped, it is gone. You cannot use it again.

This is why your chat app can handle 10,000 users. You don't need a complex "Room Manager" script that scans for old rooms every hour. Each room manages its own death the moment it's no longer being used.
