``` 

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

## in this function `create_room_handler` you are spawning threads. will not the threads be died after the function call?

That is a great question, and it's one of the most powerful features of **Tokio**. The short answer is: **No, they will not die.**

In standard synchronous Rust, if you use `std::thread::spawn`, the thread continues running even after the function returns. However, in asynchronous Rust, `tokio::spawn` works slightly differently but follows the same principle of "detachment."



### 1. Detached Tasks
When you call `tokio::spawn`, you are creating a **Tokio Task** (often called a "Green Thread").
*   Once spawned, the task is handed over to the **Tokio Runtime** (the multithreaded scheduler running in the background).
*   The function `create_room_handler` finishes and returns a response to the user immediately, but the tasks you spawned are now owned by the **Runtime**, not the function.
*   They will continue to run independently until they either finish their work, or the entire program shuts down.

### 2. The Role of `async move`
Notice the `move` keyword in `tokio::spawn(async move { ... })`. This is critical.
*   By default, closures try to borrow variables from their environment.
*   Since `create_room_handler` is about to end, any variables inside it (like `id_reap` or `state_reap`) would be destroyed.
*   The `move` keyword forces the task to **take ownership** of those variables. It physically moves them into the task's memory space so they stay alive as long as the task needs them, even after the original function is long gone.

### 3. What actually "dies"?
The only thing that "dies" when the function returns is the **JoinHandle**.
`tokio::spawn` actually returns a value:
```rust
let handle = tokio::spawn(async { ... });
```
If you don't save that `handle` (which we aren't in your code), you simply lose the ability to `await` the task or cancel it manually from outside. The task itself keeps running happily in the background.

### 4. Lifecycle in your specific code:
1.  **The Reaper Task:** Lives for 10 minutes (due to `sleep`). After 10 minutes, it runs its logic and then naturally exits/dies.
2.  **The Creator Task:** Lives as long as the `mpsc::Receiver (rx)` is open. If the `Sender` in the `Room` struct is dropped (because the room was deleted), `rx.recv()` will return `None`, the `while` loop will end, and the task will then die.

### Summary
You can think of `tokio::spawn` as "fire and forget." The function finishes and moves on, but the background tasks stay alive, managed by the Tokio executor, until their internal logic is complete.

Since you're managing thousands of rooms, this is exactly what you want—the API stays fast because it doesn't wait for the 10-minute timer to finish before replying to the user!
