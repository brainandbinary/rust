Here's a complete list of the most commonly used functions in **`tokio::fs`** with examples. These are the **asynchronous** versions of the standard `std::fs` functions, useful for non-blocking operations in async Rust apps.

---

## 🔧 Prerequisites

Add `tokio` to your `Cargo.toml`:

```toml
[dependencies]
tokio = { version = "1.38", features = ["fs", "macros"] }
```

---

## ✅ File Operations

---

### 1. `tokio::fs::read(path)`

```rust
use tokio::fs;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let data = fs::read("file.txt").await?;
    println!("{:?}", data);
    Ok(())
}
```

---

### 2. `tokio::fs::read_to_string(path)`

```rust
use tokio::fs;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let contents = fs::read_to_string("file.txt").await?;
    println!("{}", contents);
    Ok(())
}
```

---

### 3. `tokio::fs::write(path, contents)`

```rust
use tokio::fs;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    fs::write("output.txt", "Hello, async!").await?;
    Ok(())
}
```

---

### 4. `tokio::fs::copy(from, to)`

```rust
use tokio::fs;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    fs::copy("file.txt", "copy.txt").await?;
    Ok(())
}
```

---

### 5. `tokio::fs::rename(from, to)`

```rust
use tokio::fs;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    fs::rename("copy.txt", "renamed.txt").await?;
    Ok(())
}
```

---

### 6. `tokio::fs::remove_file(path)`

```rust
use tokio::fs;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    fs::remove_file("unwanted.txt").await?;
    Ok(())
}
```

---

## ✅ Directory Operations

---

### 7. `tokio::fs::create_dir(path)`

```rust
use tokio::fs;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    fs::create_dir("new_dir").await?;
    Ok(())
}
```

---

### 8. `tokio::fs::create_dir_all(path)`

```rust
use tokio::fs;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    fs::create_dir_all("a/b/c").await?;
    Ok(())
}
```

---

### 9. `tokio::fs::remove_dir(path)`

```rust
use tokio::fs;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    fs::remove_dir("new_dir").await?;
    Ok(())
}
```

---

### 10. `tokio::fs::remove_dir_all(path)`

```rust
use tokio::fs;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    fs::remove_dir_all("a").await?;
    Ok(())
}
```

---

### 11. `tokio::fs::read_dir(path)`

```rust
use tokio::fs;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let mut entries = fs::read_dir(".").await?;

    while let Some(entry) = entries.next_entry().await? {
        println!("{:?}", entry.path());
    }

    Ok(())
}
```

---

## ✅ Metadata and Permissions

---

### 12. `tokio::fs::metadata(path)`

```rust
use tokio::fs;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let metadata = fs::metadata("file.txt").await?;
    println!("Size: {}", metadata.len());
    Ok(())
}
```

---

### 13. `tokio::fs::symlink_metadata(path)`

Similar to `metadata`, but does **not follow symlinks**.

---

### 14. `tokio::fs::canonicalize(path)`

```rust
use tokio::fs;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let absolute_path = fs::canonicalize("file.txt").await?;
    println!("{:?}", absolute_path);
    Ok(())
}
```

---

### 15. `tokio::fs::set_permissions(path, permissions)`

```rust
use tokio::fs;
use std::fs::Permissions;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let perms = Permissions::from_mode(0o644); // use `std::os::unix::fs::PermissionsExt` for Unix
    fs::set_permissions("file.txt", perms).await?;
    Ok(())
}
```

---

## ✅ Working with Files: `tokio::fs::File`

```rust
use tokio::fs::File;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let mut file = File::create("data.txt").await?;
    file.write_all(b"Async file IO").await?;

    let mut file = File::open("data.txt").await?;
    let mut contents = vec![];
    file.read_to_end(&mut contents).await?;
    println!("{:?}", String::from_utf8_lossy(&contents));
    Ok(())
}
```

---

## 🧠 Summary Table

| Function                       | Purpose                            |
| ------------------------------ | ---------------------------------- |
| `read`                         | Read file as `Vec<u8>`             |
| `read_to_string`               | Read file as `String`              |
| `write`                        | Write bytes to a file              |
| `copy`                         | Copy file                          |
| `rename`                       | Rename/move file                   |
| `remove_file`                  | Delete a file                      |
| `create_dir`, `create_dir_all` | Create directory                   |
| `remove_dir`, `remove_dir_all` | Remove directory                   |
| `read_dir`                     | Iterate over directory entries     |
| `metadata`, `symlink_metadata` | Get file metadata                  |
| `canonicalize`                 | Get absolute path                  |
| `set_permissions`              | Change permissions                 |
| `File::open/create`            | Read/write files with more control |

---

To **write to a binary file** using `tokio::fs`, you can use the `tokio::fs::File` type along with the `AsyncWriteExt` trait from `tokio::io`.

---

## ✅ Step-by-Step Example: Writing to a Binary File

```rust
use tokio::fs::File;
use tokio::io::AsyncWriteExt;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    // Open or create the binary file
    let mut file = File::create("output.bin").await?;

    // Example binary data (e.g., u8 values)
    let buffer: Vec<u8> = vec![0xDE, 0xAD, 0xBE, 0xEF];

    // Write binary data to the file
    file.write_all(&buffer).await?;

    println!("Binary data written to output.bin");
    Ok(())
}
```

---

## 📦 If you want to write structured binary data (e.g., integers, structs), you can use `bincode` or `byteorder`.

### 🔁 Example: Writing a `u32` in little-endian using `byteorder`

Add dependencies in `Cargo.toml`:

```toml
[dependencies]
tokio = { version = "1.38", features = ["fs", "macros"] }
byteorder = "1.5"
```

Then:

```rust
use tokio::fs::File;
use tokio::io::{AsyncWriteExt, BufWriter};
use byteorder::{LittleEndian, WriteBytesExt};

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let file = File::create("number.bin").await?;
    let mut writer = BufWriter::new(file);

    // Writing a 32-bit unsigned integer (12345678) in little-endian
    writer.write_u32::<LittleEndian>(12345678).await?;

    println!("u32 written in little-endian format.");
    Ok(())
}
```

> 🔎 `BufWriter` helps with performance by reducing the number of actual system calls.

---

## 📚 Summary

| Task                      | Tool / Trait           |
| ------------------------- | ---------------------- |
| Basic binary write        | `File + write_all()`   |
| Struct/typed binary write | `byteorder`, `bincode` |
| Buffered write            | `BufWriter`            |

Sure! Here's how you can **serialize a JSON object** (e.g., a struct or `serde_json::Value`) into a **binary file** using `Tokio` and write it as **binary data**:

---

## ✅ Goal: Serialize JSON → Write to Binary File

### 🧱 Requirements:

In `Cargo.toml`, add:

```toml
[dependencies]
tokio = { version = "1.38", features = ["fs", "macros"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

---

## 🧪 Example: Serialize Struct to JSON → Write to Binary File

```rust
use tokio::fs::File;
use tokio::io::AsyncWriteExt;
use serde::{Serialize, Deserialize};
use serde_json;

#[derive(Serialize, Deserialize)]
struct User {
    id: u32,
    username: String,
    email: String,
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    // Create sample data
    let user = User {
        id: 1,
        username: "async_rustacean".to_string(),
        email: "user@example.com".to_string(),
    };

    // Convert struct to JSON string
    let json_string = serde_json::to_string(&user).unwrap();

    // Convert string to bytes
    let json_bytes = json_string.as_bytes();

    // Create or open a binary file
    let mut file = File::create("user_data.bin").await?;

    // Write JSON bytes into the binary file
    file.write_all(json_bytes).await?;

    println!("JSON written to binary file as bytes.");
    Ok(())
}
```

---

## 📂 Output

The `user_data.bin` file will contain:

```json
{"id":1,"username":"async_rustacean","email":"user@example.com"}
```

But in **binary format** (i.e., raw UTF-8 bytes).

---

## 🔁 Bonus: Read it back later

```rust
use tokio::fs::read;
use serde_json::from_slice;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let bytes = read("user_data.bin").await?;
    let user: User = from_slice(&bytes).unwrap();

    println!("Read from binary: {} ({})", user.username, user.email);
    Ok(())
}
```

---

Great! Here's how you can write **compressed binary JSON** using [`bincode`](https://docs.rs/bincode/) — which is **more compact and faster** than text-based JSON.

---

## ✅ Goal: Serialize a Rust struct with `bincode` → Write binary file

### 🧱 Dependencies in `Cargo.toml`:

```toml
[dependencies]
tokio = { version = "1.38", features = ["fs", "macros"] }
serde = { version = "1.0", features = ["derive"] }
bincode = "1.3"
```

---

## 🧪 Example: Writing a Struct to a Binary File Using `bincode`

```rust
use tokio::fs::File;
use tokio::io::AsyncWriteExt;
use serde::{Serialize, Deserialize};
use bincode;

#[derive(Serialize, Deserialize, Debug)]
struct User {
    id: u32,
    username: String,
    email: String,
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    // Create a sample user
    let user = User {
        id: 1,
        username: "compressed_rustacean".to_string(),
        email: "user@bincode.io".to_string(),
    };

    // Serialize to binary using bincode
    let encoded: Vec<u8> = bincode::serialize(&user).unwrap();

    // Write binary data to a file
    let mut file = File::create("user_data_compressed.bin").await?;
    file.write_all(&encoded).await?;

    println!("Bincode binary data written to file.");
    Ok(())
}
```

---

## 🔁 Read from Binary File (Deserialization)

```rust
use tokio::fs::read;
use serde::{Serialize, Deserialize};
use bincode;

#[derive(Serialize, Deserialize, Debug)]
struct User {
    id: u32,
    username: String,
    email: String,
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let data = read("user_data_compressed.bin").await?;
    let user: User = bincode::deserialize(&data).unwrap();

    println!("Read user: {:?}", user);
    Ok(())
}
```

---

## 📌 Summary

| Format  | Size    | Human-readable | Fast to Parse |
| ------- | ------- | -------------- | ------------- |
| JSON    | Larger  | ✅ Yes          | ❌ Slower      |
| Bincode | Compact | ❌ No           | ✅ Fast        |

---

Perfect! Here's how you can **serialize and write multiple structs to a single binary file** using [`bincode`](https://docs.rs/bincode/) — and later **read them back one by one** like a stream.

This is especially useful for logging, data export, or building your own compact database format.

---

## 🧱 Dependencies (`Cargo.toml`)

```toml
[dependencies]
tokio = { version = "1.38", features = ["fs", "macros"] }
serde = { version = "1.0", features = ["derive"] }
bincode = "1.3"
tokio-util = { version = "0.7", features = ["codec"] }
```

---

## ✍️ Writing Multiple Structs to a Binary File

```rust
use tokio::fs::File;
use tokio::io::AsyncWriteExt;
use serde::{Serialize, Deserialize};
use bincode;

#[derive(Serialize, Deserialize, Debug)]
struct LogEntry {
    timestamp: u64,
    message: String,
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let mut file = File::create("logs.bin").await?;

    let logs = vec![
        LogEntry { timestamp: 1690000000, message: "System started".into() },
        LogEntry { timestamp: 1690000100, message: "User logged in".into() },
        LogEntry { timestamp: 1690000200, message: "Error: disk full".into() },
    ];

    for entry in logs {
        let bytes = bincode::serialize(&entry).unwrap();

        // Optional: prefix each message with its length (helps deserialization later)
        let len_bytes = (bytes.len() as u32).to_le_bytes(); // 4-byte length header
        file.write_all(&len_bytes).await?;
        file.write_all(&bytes).await?;
    }

    println!("Log entries written as stream.");
    Ok(())
}
```

---

## 📖 Reading Multiple Structs from Binary File

```rust
use tokio::fs::File;
use tokio::io::{AsyncReadExt, BufReader};
use serde::{Serialize, Deserialize};
use bincode;

#[derive(Serialize, Deserialize, Debug)]
struct LogEntry {
    timestamp: u64,
    message: String,
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let file = File::open("logs.bin").await?;
    let mut reader = BufReader::new(file);

    loop {
        let mut len_buf = [0u8; 4];

        // Read the 4-byte length header
        if let Err(_) = reader.read_exact(&mut len_buf).await {
            break; // EOF
        }

        let len = u32::from_le_bytes(len_buf) as usize;
        let mut data_buf = vec![0u8; len];

        // Read the binary-encoded struct
        reader.read_exact(&mut data_buf).await?;

        let log: LogEntry = bincode::deserialize(&data_buf).unwrap();
        println!("Read log: {:?}", log);
    }

    Ok(())
}
```

---

## 🧠 Notes

* We prefix each binary-encoded struct with a 4-byte length. This allows the reader to know how many bytes to read next.
* This pattern is efficient and works well even with huge files.

---

Awesome — let’s build a simple **Tokio TCP server and client** that stream multiple binary-encoded structs (e.g., logs or events) using `bincode`.

---

## ✅ Goal

* 📤 Server: Sends multiple structs over a TCP stream.
* 📥 Client: Reads them one-by-one using length-prefixed binary messages.

We'll reuse the `LogEntry` struct and prefix each serialized struct with its 4-byte length (same as we did for file).

---

## 🧱 Dependencies (`Cargo.toml`)

```toml
[dependencies]
tokio = { version = "1.38", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
bincode = "1.3"
```

---

## 🧩 Shared Struct

```rust
// src/common.rs
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize, Debug)]
pub struct LogEntry {
    pub timestamp: u64,
    pub message: String,
}
```

---

## 🖥️ Server (sends multiple structs)

```rust
// src/bin/server.rs
use tokio::net::TcpListener;
use tokio::io::AsyncWriteExt;
use bincode;
use std::error::Error;

mod common;
use common::LogEntry;

#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    println!("Server running on 127.0.0.1:8080");

    loop {
        let (mut socket, _) = listener.accept().await?;

        let logs = vec![
            LogEntry { timestamp: 1690000000, message: "System started".into() },
            LogEntry { timestamp: 1690000100, message: "User logged in".into() },
            LogEntry { timestamp: 1690000200, message: "Disk error".into() },
        ];

        for entry in logs {
            let data = bincode::serialize(&entry)?;
            let len = (data.len() as u32).to_le_bytes();

            socket.write_all(&len).await?;
            socket.write_all(&data).await?;
        }

        println!("Logs sent.");
    }
}
```

---

## 💻 Client (reads structs from the stream)

```rust
// src/bin/client.rs
use tokio::net::TcpStream;
use tokio::io::{AsyncReadExt, BufReader};
use bincode;
use std::error::Error;

mod common;
use common::LogEntry;

#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    let stream = TcpStream::connect("127.0.0.1:8080").await?;
    let mut reader = BufReader::new(stream);

    loop {
        let mut len_buf = [0u8; 4];

        if reader.read_exact(&mut len_buf).await.is_err() {
            break; // EOF
        }

        let len = u32::from_le_bytes(len_buf) as usize;
        let mut data_buf = vec![0u8; len];
        reader.read_exact(&mut data_buf).await?;

        let log: LogEntry = bincode::deserialize(&data_buf)?;
        println!("Received log: {:?}", log);
    }

    Ok(())
}
```

---

## 🧪 Run it

1. **Start the server**

   ```
   cargo run --bin server
   ```

2. **In another terminal, start the client**

   ```
   cargo run --bin client
   ```

---

## ✅ Summary

| Feature                   | Implementation                                         |
| ------------------------- | ------------------------------------------------------ |
| Efficient binary transfer | [`bincode`](https://docs.rs/bincode/)                  |
| Async networking          | [`tokio::net`](https://docs.rs/tokio/latest/tokio/net) |
| Length-prefixed streaming | 4-byte `u32` header before each struct                 |

---





