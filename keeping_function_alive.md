## Clear me, when will the block be called, 1) on function handle_socket call? 2) on function handle_socket die? extra) will the function live as long as the socket open? or the function remains blocked by while loop in the thread, when the thread will be finished the function will die?
```
    // --- 4. Dynamic Cleanup ---
    if let Some(mut room) = state.room_list.get_mut(&room_id) {
        let mut survivor_tx: Option<mpsc::Sender<String>> = None;

        if is_user_1 {
            room.user_1_tx = None;
            survivor_tx = room.user_2_tx.clone(); // If User 1 leaves, look up User 2 dynamically
        } else {
            room.user_2_tx = None;
            survivor_tx = room.user_1_tx.clone(); // If User 2 leaves, look up User 1 dynamically
        }

        // Notify the remaining user that their partner dropped out
        if let Some(peer_tx) = survivor_tx {
            let _ = peer_tx.send(r#"{"action": "peer_left"}"#.to_string()).await;
        }

        // Drop the room completely if nobody is left
        if room.user_1_tx.is_none() && room.user_2_tx.is_none() {
            println!("before drop the room {:#?} :", room);
            drop(room); // Avoid deadlock prior to map removal
            let event = DashboardEvent::RoomDeleted {
                room_id: room_id.clone(),
            };
            let _ = state.dashboard_tx.send(event);
            state.room_list.remove(&room_id);
        }
    }
```
###In  handle_socket
```
  pub async fn websocket_handler(
    ws: WebSocketUpgrade,
    Path(room_id): Path<String>,
    State(state): State<Arc<AppState>>,
) -> impl IntoResponse {
    ws.on_upgrade(move |socket| handle_socket(socket, state, room_id))
}

async fn handle_socket(socket: WebSocket, state: Arc<AppState>, room_id: String) {
    let (mut ws_sender, mut ws_receiver) = socket.split();
    let (internal_tx, mut internal_rx) = mpsc::channel::<String>(100);

    let mut is_user_1 = false;
    let mut peer_to_notify: Option<mpsc::Sender<String>> = None;

    // --- 1. Registration & Role Assignment ---
    {
        let mut room = match state.room_list.get_mut(&room_id) {
            Some(r) => r,
            None => {
                let _ = ws_sender.send(Message::Text("Room not found".into())).await;
                return;
            }
        };

        if room.user_1_tx.is_none() {
            room.user_1_tx = Some(internal_tx);
            is_user_1 = true;

            // If User 2 was ALREADY here waiting, User 1 must notify User 2
            if room.user_2_tx.is_some() {
                peer_to_notify = room.user_2_tx.clone();
            }
        } else if room.user_2_tx.is_none() {
            room.user_2_tx = Some(internal_tx);
            is_user_1 = false;

            // User 1 was already here, User 2 must notify User 1
            peer_to_notify = room.user_1_tx.clone();
        } else {
            let _ = ws_sender
                .send(Message::Text(r#"{"action": "room_full"}"#.into()))
                .await;
            return;
        }
    }

    // Trigger WebRTC offer creation on the frontend client side
    if let Some(peer_tx) = peer_to_notify {
        let _ = peer_tx
            .send(r#"{"action": "peer_joined"}"#.to_string())
            .await;
    }

    // --- 2. Outbound Task (KISS: Keep It Simple, Stupid) ---
    // This task should have ONE job: forward messages from the channel to the browser.
    let mut send_task = tokio::spawn(async move {
        while let Some(msg) = internal_rx.recv().await {
            if ws_sender.send(Message::Text(msg.into())).await.is_err() {
                break;
            }
        }
    });

    // --- 3. Inbound Task ---
    let state_clone = state.clone();
    let room_id_clone = room_id.clone();

    let mut recv_task = tokio::spawn(async move {
        while let Some(msg_result) = ws_receiver.next().await {
            match msg_result {
                Ok(Message::Text(text)) => {
                    if let Some(room) = state_clone.room_list.get(&room_id_clone) {
                        let target_peer = if is_user_1 {
                            &room.user_2_tx
                        } else {
                            &room.user_1_tx
                        };
                        if let Some(peer_tx) = target_peer {
                            let _ = peer_tx.send(text.to_string()).await;
                        }
                    }
                }
                Ok(Message::Close(_)) => break,
                Err(_) => break,
                _ => {} // Safely skip binary, ping, and pong frames
            }
        }
    });

    // Run until one task finishes or aborts
    tokio::select! {
        _ = (&mut send_task) => recv_task.abort(),
        _ = (&mut recv_task) => send_task.abort(),
    };

    // --- 4. Dynamic Cleanup ---
    if let Some(mut room) = state.room_list.get_mut(&room_id) {
        let mut survivor_tx: Option<mpsc::Sender<String>> = None;

        if is_user_1 {
            room.user_1_tx = None;
            survivor_tx = room.user_2_tx.clone(); // If User 1 leaves, look up User 2 dynamically
        } else {
            room.user_2_tx = None;
            survivor_tx = room.user_1_tx.clone(); // If User 2 leaves, look up User 1 dynamically
        }

        // Notify the remaining user that their partner dropped out
        if let Some(peer_tx) = survivor_tx {
            let _ = peer_tx.send(r#"{"action": "peer_left"}"#.to_string()).await;
        }

        // Drop the room completely if nobody is left
        if room.user_1_tx.is_none() && room.user_2_tx.is_none() {
            println!("before drop the room {:#?} :", room);
            drop(room); // Avoid deadlock prior to map removal
            let event = DashboardEvent::RoomDeleted {
                room_id: room_id.clone(),
            };
            let _ = state.dashboard_tx.send(event);
            state.room_list.remove(&room_id);
        }
    }
}


```


Let’s break down exactly how the lifecycle of this asynchronous function works. Rust’s execution flow here is highly predictable once you visualize how the runtime manages tasks.

To answer your first question directly: **The cleanup block will be called only when the `handle_socket` function is about to die (exit).**

Here is the exact step-by-step breakdown of how the function lives, blocks, and eventually reaches that cleanup code.

---

### 1. Will the function live as long as the socket is open?

**Yes.** The `handle_socket` function acts as the "manager" of the entire connection lifecycle. It stays alive precisely as long as the WebSocket connection remains open and active.

---

### 2. Is the function blocked by a while loop in a thread?

Not exactly a "thread" in the traditional sense, but **yes, execution is held in place by the `tokio::select!` block.**

Here is how that works under the hood:

* You spawn two background tasks using `tokio::spawn`: `send_task` (outbound) and `recv_task` (inbound).
* Both of these tasks contain `while let Some(...)` loops that wait for data.
* Right below them, you call `tokio::select!`. This is where the main `handle_socket` execution completely **pauses (yields control back to the Tokio runtime)**.

The function doesn't waste CPU power while waiting; it just sits at the `tokio::select!` milestone.

---

### 3. When does the thread finish and the function die?

The `tokio::select!` block is waiting for **either** `send_task` or `recv_task` to finish.

As soon as a client closes their browser tab, drops their Wi-Fi connection, or an error occurs:

1. The `ws_receiver.next()` stream yields `None` or an `Err`.
2. The `recv_task` loop breaks and the task finishes.
3. `tokio::select!` notices `recv_task` finished. It immediately wakes up and executes its matching block: it **aborts** the `send_task`.
4. Now that `tokio::select!` is completed, execution drops straight down to the next line of code.

---

### Visualizing the Lifecycle Execution Order

Here is the sequential order of your code's execution layout:

```rust
async fn handle_socket(...) {
    // [STEP A] Executed IMMEDIATELY when a user connects.
    // Assigns roles (User 1 or User 2) and saves senders to state.
    { ... Registration & Role Assignment ... }

    // [STEP B] Spawns background workers.
    let mut send_task = tokio::spawn(...);
    let mut recv_task = tokio::spawn(...);

    // [STEP C] PAUSE ZONE. 
    // The function stays stuck right here for minutes or hours 
    // while the users are actively talking over WebRTC.
    tokio::select! {
        _ = (&mut send_task) => recv_task.abort(),
        _ = (&mut recv_task) => send_task.abort(),
    };

    // [STEP D] CLEANUP ZONE.
    // This code ONLY runs *after* the connection has broken and 
    // tokio::select! has broken open.
    // This is when your `// --- 4. Dynamic Cleanup ---` block is called.
    if let Some(mut room) = state.room_list.get_mut(&room_id) {
        // ... Removes user from state, notifies partner, deletes room ...
    }

    // [STEP E] The function reaches its closing bracket and officially dies.
}

```

### Summary

* **When is it called?** On option **2) when `handle_socket` is dying**. It is the last thing the function does before it terminates completely.
* **Why does it wait?** Because `tokio::select!` keeps the function execution suspended in mid-air right above the cleanup block for the entire duration of the call.
