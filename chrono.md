Sure! Here’s a **summary of commonly used functions and types** from the Rust [`chrono`](https://docs.rs/chrono/latest/chrono/) crate, with simple examples to demonstrate how to use them.

---

# Chrono Crate — Key Functions & Examples

---

### 1. Import basics

```rust
use chrono::{DateTime, Utc, Local, NaiveDate, NaiveTime, NaiveDateTime, Duration, TimeZone};
```

---

### 2. Getting current time

```rust
let now_utc: DateTime<Utc> = Utc::now();
println!("Current UTC time: {}", now_utc);

let now_local: DateTime<Local> = Local::now();
println!("Current Local time: {}", now_local);
```

---

### 3. Creating dates and times

* **NaiveDate** (date without timezone)

```rust
let date = NaiveDate::from_ymd(2025, 7, 28);
println!("NaiveDate: {}", date);
```

* **NaiveTime** (time without date or timezone)

```rust
let time = NaiveTime::from_hms(14, 30, 0);
println!("NaiveTime: {}", time);
```

* **NaiveDateTime** (date + time without timezone)

```rust
let datetime = NaiveDateTime::new(date, time);
println!("NaiveDateTime: {}", datetime);
```

---

### 4. Parsing from strings

```rust
// Parsing RFC3339 datetime string with timezone
let dt = DateTime::parse_from_rfc3339("2025-07-28T14:30:00+02:00").unwrap();
println!("Parsed DateTime: {}", dt);

// Parsing naive date from string with format
let nd = NaiveDate::parse_from_str("2025-07-28", "%Y-%m-%d").unwrap();
println!("Parsed NaiveDate: {}", nd);
```

---

### 5. Formatting dates/times

```rust
let now = Utc::now();

// Format as YYYY-MM-DD HH:MM:SS
println!("Formatted date: {}", now.format("%Y-%m-%d %H:%M:%S"));

// Format with weekday name
println!("Formatted with weekday: {}", now.format("%A, %B %d, %Y"));
```

---

### 6. Adding and subtracting durations

```rust
let now = Utc::now();
let later = now + Duration::days(5);
let earlier = now - Duration::hours(3);

println!("5 days later: {}", later);
println!("3 hours earlier: {}", earlier);
```

---

### 7. Working with durations

```rust
let dur = Duration::seconds(3600);  // 1 hour
println!("Duration in seconds: {}", dur.num_seconds());
println!("Duration in minutes: {}", dur.num_minutes());
```

---

### 8. Timezone conversions

```rust
let utc_time = Utc::now();
let local_time = utc_time.with_timezone(&Local);
println!("UTC time: {}", utc_time);
println!("Local time: {}", local_time);
```

---

### 9. Extracting date/time components

```rust
let now = Utc::now();

println!("Year: {}", now.year());
println!("Month: {}", now.month());
println!("Day: {}", now.day());
println!("Hour: {}", now.hour());
println!("Minute: {}", now.minute());
println!("Second: {}", now.second());
```

---

### 10. Creating `Date` from `NaiveDate` and timezone

```rust
let naive_date = NaiveDate::from_ymd(2025, 7, 28);
let date = Utc.from_utc_date(&naive_date);
println!("Date with timezone: {}", date);
```

---

### 11. Getting timestamp and converting from timestamp

```rust
let now = Utc::now();
let timestamp = now.timestamp();
println!("Unix timestamp: {}", timestamp);

let dt_from_ts = Utc.timestamp(timestamp, 0);
println!("DateTime from timestamp: {}", dt_from_ts);
```

---

### 12. Comparing dates/times

```rust
let dt1 = Utc::now();
let dt2 = dt1 + Duration::days(1);

println!("dt1 < dt2? {}", dt1 < dt2);
println!("dt1 == dt2? {}", dt1 == dt2);
```

---

### 13. Extracting weekday

```rust
let now = Utc::now();
println!("Weekday: {}", now.weekday());  // e.g., "Mon"
```

---

### Summary Table of Common Types and Their Methods

| Type            | Purpose                        | Example Method                                        |
| --------------- | ------------------------------ | ----------------------------------------------------- |
| `DateTime<Tz>`  | Date & time with timezone      | `now()`, `format()`, `timestamp()`, `with_timezone()` |
| `NaiveDate`     | Date without timezone          | `from_ymd()`, `parse_from_str()`, `year()`, `month()` |
| `NaiveTime`     | Time without timezone          | `from_hms()`, `hour()`, `minute()`                    |
| `NaiveDateTime` | Date & time without timezone   | `new()`, `parse_from_str()`                           |
| `Duration`      | Time duration (interval)       | `days()`, `seconds()`, `num_minutes()`                |
| `TimeZone`      | Trait for timezone conversions | `from_utc_date()`, `timestamp()`                      |

---

