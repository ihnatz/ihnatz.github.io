---
layout: post
title:  "Fast PostgreSQL Data Generation"
date:   2024-11-07 22:18:11
categories: postgres rust
---

Improving query performance is always challenging, especially when working with large-scale databases. While adjusting indexes is a common approach, testing these optimizations becomes complex when your tables contain terabytes of data. I recently faced this challenge: I needed to test potential improvements for a large-scale table, but there was no simple way to work with terabytes of data on my laptop. The solution? Generating a representative dataset that mimics our production database structure and patterns.

In this article, I'll explore several approaches to generating and inserting large volumes of test data into PostgreSQL on a local machine. While many of these techniques can be applied to production environments and other databases, we'll focus specifically on local development with PostgreSQL.

__Note__: All benchmarks in this article were run on a modest laptop - you'll likely see much better performance on modern hardware. The focus here is on relative improvements between different approaches rather than absolute numbers.

## Table structure

We'll work with a simple time-series-like table containing three fields, no indexes at the moment to keep it simple:

```sql
CREATE TABLE metrics (
    created     timestamp with time zone default now() NOT NULL,
    sensor_id   integer                                NOT NULL,
    temperature numeric                                NOT NULL
);
```

## Basic case

The most straightforward approach is to leverage PostgreSQL's built-in capabilities for data generation. Let's start with a simple SQL script that generates 10 million rows:

```sql
INSERT INTO metrics (
    SELECT
        (NOW() + seq / 100 * interval '1 MILLISECOND') AS created,
        (seq % 32 + 1)::integer AS sensor_id,
        20.0 + RANDOM() * 500 / 100 as temperature
    FROM GENERATE_SERIES(1, 10_000_000) seq
);
```

This script:

- Creates timestamps at 10μs intervals starting from now
- Simulates 32 different sensors (IDs 1-32)
- Generates temperatures between 20.0°C and 25.0°C

On my local machine this query takes **26.440s** which is quite fast for 10 million rows (~0.5GB of data). This is probably the best case, since it eliminates overhead from connection handling and data type conversions - PostgreSQL handles everything internally.

## Generating actual data

In our production database, sensor readings arrive in batches: multiple sensors record values at the exact same timestamp. This pattern has important implications for database performance, particularly for indexing and query optimization. Let's implement a Rust generator that mimics this behavior:

```rust
const BATCH_SIZE: usize = 1_000; // Number of sensor readings per batch
const MAX_SENSORS: i32 = 32; // Total number of unique sensors

type Row = (DateTime<Utc>, i32, f64);

fn generate_batch(created: DateTime<Utc>, sensor_id: i32, base_temp: f64) -> (Vec<Row>, i32) {
    let mut rng = rand::thread_rng();
    let mut current_sensor_id = sensor_id;
    let batch: Vec<_> = (0..BATCH_SIZE)
        .map(|i| {
            current_sensor_id = (current_sensor_id + (i as i32)) % MAX_SENSORS + 1;
            let temperature = ((base_temp + rng.gen_range(-5.0..5.0)) * 100.0).round() / 100.0;
            (created, current_sensor_id, temperature)
        })
        .collect();
    (batch, sensor_id)
}

fn generate_data(
    start_time: DateTime<Utc>,
    base_temp: f64,
    batch_count: usize,
) -> impl Iterator<Item = (Vec<Row>, i64)> {
    let mut current_time = start_time;
    let mut sensor_id = 1;
    let mut current_tick = 0;

    (0..batch_count).flat_map(move |_| {
        current_tick += 1;
        current_time += Duration::milliseconds(100);
        let (new_batch, new_sensor_id) = generate_batch(current_time, sensor_id, base_temp);
        sensor_id = new_sensor_id;

        std::iter::once((new_batch, current_tick))
    })
}
```

This generator creates data that closely mirrors our production patterns:
- Each batch generates 1,000 readings sharing the same timestamp
- Multiple readings from each sensor (ID range 1-32) within each batch
- Batches are generated at 100ms intervals
- Temperatures vary randomly within ±5°C of the base temperature
- Each reading is rounded to 2 decimal places

While this is a simplified version of our production data patterns, it captures the key characteristics that affect query performance: time clustering, sensor ID distribution, and value ranges.

## Measuring performance

To effectively compare different data insertion approaches, we need consistent performance metrics. Let's create a helper struct that measures execution time, data size, and throughput using RAII (Resource Acquisition Is Initialization) pattern:

```rust
struct ExecutionContext {
    t0: DateTime<Utc>,
    s0: i64,
    client: Client,
    name: String,
}

impl ExecutionContext {
    fn new(name: &str, conn_info: &str) -> Self {
        let mut client = Client::connect(conn_info, NoTls).unwrap();
        let s0 = Self::table_size(&mut client);
        let t0 = Utc::now();
        let name = name.to_string();

        ExecutionContext { t0, s0, client, name }
    }

    fn table_size(client: &mut Client) -> i64 {
        let row = client
            .query_one("SELECT pg_total_relation_size('public.metrics') as size", &[])
            .unwrap();
        row.get("size")
    }

    fn convert_bytes(bytes: f64, to: &str) -> f64 {
        let units = ["B", "KB", "MB", "GB", "TB", "PB"];
        let index = units
            .iter()
            .position(|&r| r == to.to_uppercase())
            .unwrap_or(0);
        bytes / (1024f64.powi(index as i32))
    }
}

impl Drop for ExecutionContext {
    fn drop(&mut self) {
        let s1 = Self::table_size(&mut self.client);
        let t1 = Utc::now();

        let duration = (t1 - self.t0).num_seconds();
        let size = s1 - self.s0;
        let speed = (size as f64) / (duration as f64);

        println!("\n{}:", self.name);
        println!("Speed: {:.2}MB/s", Self::convert_bytes(speed, "MB"));
        println!(" Data: {:.2}MB", Self::convert_bytes(size as f64, "MB"));
        println!(" Time: {:.2}s", duration);
    }
}
```

This performance measuring tool:
1. Tracks the start time and initial table size when created
2. Maintains a database connection throughout the test
3. Automatically calculates and prints performance metrics when dropped

Here's how to use it:

```rust
{
    let _context = ExecutionContext::new("Simple Insert Test", conn_info);
    let stmt = "insert into metrics values (NOW(), (RANDOM() * 20)::integer, 20 + RANDOM())";
    for _ in 0..1_000 {
        client.execute(stmt, &[]).unwrap();
    }
}

// Output:
// Simple Insert Test:
// Speed: 0.01MB/s
// Data: 0.06MB
// Time: 5s
```

This tool will help us compare the performance of different data insertion strategies in the following sections.

## Simple insert

Let's start with the most straightforward approach: inserting records one by one within batch transactions. While this isn't the most efficient method, it provides a baseline for our performance improvements.

```rust
fn f64_to_decimal(value: f64) -> Decimal {
    Decimal::from_str(&value.to_string()).unwrap_or_else(|_| Decimal::new(0, 0))
}

fn insert_to_postgres(
    client: &mut Client,
    table_name: &str,
    batch_data: &[Row],
    current_tick: i64,
) {
    let mut tx = client.transaction().unwrap();
    let stmt = tx
        .prepare(&format!("INSERT INTO {} VALUES ($1, $2, $3)", table_name))
        .unwrap();

    for row in batch_data {
        // PostgreSQL's decimal type requires explicit conversion from f64
        let params: [&(dyn ToSql + Sync); 3] = [&row.0, &row.1, &f64_to_decimal(row.2)];
        tx.execute(&stmt, &params).unwrap();
    }

    tx.commit().unwrap();

    if current_tick % REPORT_COUNT == 0 {
        println!("Copied {current_tick}");
    }
}
```

Running this with batches of 10,000 records each:

```
fn insert:
Speed: 0.89MB/s
 Data: 497.76MB
 Time: 560s
```

Compared to our initial pure SQL approach (26 seconds), this method is over 20 times slower. The performance gap comes primarily from network roundtrips - each insert requires:
1. Sending the execute command to PostgreSQL
2. PostgreSQL processing it
3. Getting the acknowledgment back

In the next sections, we'll explore how to optimize these bottlenecks.

## Long string insert

While network latency is typically low on local connections, the overhead of making thousands of separate requests adds up significantly. Instead, we can batch our inserts into a single SQL statement. This approach:
- Combines all values into one INSERT command
- Lets PostgreSQL handle data type conversions
- Minimizes network roundtrips

```rust
fn insert_to_postgres_string(
    client: &mut Client,
    table_name: &str,
    batch_data: &[Row],
    current_tick: i64,
) {
    let tuples = batch_data
        .into_iter()
        .map(|row| {
            format!(
                "('{}'::timestamp with time zone, {}, {}::numeric(10, 2))",
                row.0, row.1, row.2
            )
        })
        .collect::<Vec<_>>()
        .join(",");
    let query = format!("INSERT INTO {} VALUES {}", table_name, tuples);
    client.execute(&query, &[]).unwrap();

    if current_tick % REPORT_COUNT == 0 {
        println!("Copied {current_tick}");
    }
}

```

This generates a query that looks like:

```sql
INSERT INTO metrics VALUES
('2024-01-01 10:00:00+00'::timestamp with time zone, 1, 20.5::numeric(10, 2)),
('2024-01-01 10:00:00+00'::timestamp with time zone, 2, 21.3::numeric(10, 2)),
...
```


Rather than sending 10,000 separate INSERT commands, we send one large query. While the resulting SQL isn't as readable as individual INSERT statements, it's much more efficient to transmit and execute. Let's check how it performs:

```
fn insert-str:
Speed: 4.48MB/s
 Data: 497.75MB
 Time: 111s
```

By eliminating multiple requests and sending just one, we achieved a 5x speed improvement - quite impressive for such a straightforward change.

## Binary COPY Protocol

Our string-based approach still has overhead: generating SQL strings and letting PostgreSQL parse them. Let's go further and speak the native language of the database: binary data. PostgreSQL provides this capability through the `COPY` command.

From PostgreSQL documentation:
> COPY FROM copies data from a file to a table (appending the data to whatever is in the table already)

The binary format option is particularly interesting:
> The binary format option causes all data to be stored/read as binary format rather than as text. It is somewhat faster than the text and CSV formats, but a binary-format file is less portable across machine architectures and PostgreSQL versions.

Why is this relevant to us? We generate test data locally, so portability isn't an issue. Instead, we can focus purely on performance by
- Eliminating the overhead of string formatting
- Reducing the amount of data transferred
- Allowing PostgreSQL to skip the SQL parsing step

The binary format protocol requires careful handling of different data types:
- timestamp with time zone
- integer
- numeric (decimal)

Rust makes this task easy - it's both convenient for handling structured data and efficient at manipulating raw bytes. Let's implement the binary encoding of each type, from the simplest to the most complex.

### integer

Converting integers is really simple since both Rust's `i32` and PostgreSQL's `integer` use the same representation. All we need to do is to say how long (pun intended, as PostgreSQL also has a `long` type) the value is and write it as bytes:

```rust
// sensor_id
buffer.write_i32::<BigEndian>(4)?;
buffer.write_i32::<BigEndian>(row.1)?;
```

### timestamp with time zone

This one is more complex. How should we represent a timestamp? ISO format? Seconds since UNIX epoch?

PostgreSQL actually uses its own epoch (2000-01-01) and counts microseconds from there. Here's how we handle it:

```rust
static POSTGRES_EPOCH: Lazy<DateTime<Utc>> = Lazy::new(|| {
    NaiveDate::from_ymd_opt(2000, 1, 1)
        .unwrap()
        .and_hms_opt(0, 0, 0)
        .unwrap()
        .and_utc()
});

fn datetime_to_postgres_binary(datetime: DateTime<Utc>) -> i64 {
    let time_delta = datetime - POSTGRES_EPOCH.to_utc();
    time_delta.num_microseconds().unwrap()
}

// created
let micros = datetime_to_postgres_binary(row.0);
buffer.write_i32::<BigEndian>(8)?;
buffer.write_i64::<BigEndian>(micros)?;
```

### decimal

This is the most challenging part of our binary implementation. While PostgreSQL's binary format can be very efficient, some data types (like numeric/decimal) lack clear documentation about their binary representation. This reflects a common challenge in database development - sometimes you need to dig deeper than the official docs.

To figure out the correct implementation, we need to:
1. Read the PostgreSQL source code: [numeric.c](https://github.com/postgres/postgres/blob/REL_14_STABLE/src/backend/utils/adt/numeric.c#L944)
2. Look at existing implementations in other languages, like this [Python example](https://github.com/altaurog/pgcopy/blob/master/pgcopy/copy.py#L63)

From these sources, we can determine that each numeric value consists of:
- ndigits: number of digits in base NBASE (which is 1000)
- weight: weight of the first digit
- sign: 0x0000 for positive, 0x4000 for negative
- dscale: number of decimal digits after the decimal point

Here's our implementation:

```rust
fn numeric_to_postgres_binary(value: f64) -> Vec<u8> {
    let mut buffer = Vec::new();
    let abs_value = value.abs();
    let sign = if value.is_sign_negative() {
        0x4000
    } else {
        0x0000
    };
    let parts: Vec<String> = abs_value.to_string().split('.').map(String::from).collect();

    let integer = &parts[0];
    let fraction = if parts.len() > 1 {
        parts[1].clone()
    } else {
        String::new()
    };

    let weight = ((integer.len() as i16 - 1) / 4) as i16;
    let padding = if integer.len() % 4 != 0 {
        4 - (integer.len() % 4)
    } else {
        0
    };
    let padded_number = format!(
        "{:0>width$}{}",
        integer,
        fraction,
        width = integer.len() + padding
    );

    let digits: Vec<i16> = padded_number
        .as_bytes()
        .chunks(4)
        .map(|c| {
            let mut value = 0;
            for (i, &digit) in c.iter().enumerate() {
                value += (digit - b'0') as i16 * 10i16.pow(3 - i as u32);
            }
            value as i16
        })
        .collect();
    let ndigits = digits.len() as i16;
    let dscale = fraction.len() as i16;

    buffer.write_i16::<BigEndian>(ndigits).unwrap();
    buffer.write_i16::<BigEndian>(weight).unwrap();
    buffer.write_i16::<BigEndian>(sign).unwrap();
    buffer.write_i16::<BigEndian>(dscale).unwrap();

    for digit in digits {
        buffer.write_i16::<BigEndian>(digit).unwrap();
    }

    buffer
}

// temperature
let numeric_bytes = numeric_to_postgres_binary(row.2);
buffer.write_i32::<BigEndian>(numeric_bytes.len() as i32)?;
buffer.write_all(&numeric_bytes)?;
```

This implementation handles our decimal temperature values by converting them into PostgreSQL's internal numeric representation. The process is more complex than for integers or timestamps, but the resulting binary format is exactly what PostgreSQL expects.

### Putting It All Together

Now let's combine everything into a working solution. We'll use Rust's `Cursor` to build an in-memory buffer that follows PostgreSQL's binary COPY protocol requirements:
- A magic header to identify the format
- Number of fields per row (3 in our case)
- The actual data
- A termination marker

```rust
fn generate_buffer(batch_data: &[Row]) -> anyhow::Result<Vec<u8>> {
    let mut buffer = Cursor::new(Vec::new());
    buffer.write_all(b"PGCOPY\n\xff\r\n\0")?;
    buffer.write_i32::<BigEndian>(0)?;
    buffer.write_i32::<BigEndian>(0)?;

    for row in batch_data {
        buffer.write_i16::<BigEndian>(3)?;

        // created
        let micros = datetime_to_postgres_binary(row.0);
        buffer.write_i32::<BigEndian>(8)?;
        buffer.write_i64::<BigEndian>(micros)?;

        // sensor_id
        buffer.write_i32::<BigEndian>(4)?;
        buffer.write_i32::<BigEndian>(row.1)?;

        // temperature
        let numeric_bytes = numeric_to_postgres_binary(row.2);
        buffer.write_i32::<BigEndian>(numeric_bytes.len() as i32)?;
        buffer.write_all(&numeric_bytes)?;
    }

    buffer.write_i16::<BigEndian>(-1)?;
    Ok(buffer.into_inner())
}

fn copy_to_postgres(client: &mut Client, table_name: &str, batch_data: &[Row], current_tick: i64) {
    let buffer = generate_buffer(batch_data).unwrap();
    let mut writer = client
        .copy_in(&format!("COPY {} FROM STDIN WITH BINARY", table_name))
        .unwrap();
    writer.write_all(&buffer).unwrap();
    writer.finish().unwrap();

    if current_tick % REPORT_COUNT == 0 {
        println!("Copied {current_tick}");
    }
}
```

Yes, this implementation is more complex than our previous approaches. But did the extra complexity pay off in terms of performance? Let's find out in the next section.

### Results

Let's see if our binary implementation was worth the effort:

```
fn copy:
Speed: 24.89MB/s
 Data: 497.73MB
 Time: 20s
```

The results are impressive - 20 seconds! That's even faster than our initial raw SQL approach. Let's look at how our different approaches performed with the same data volume:

- Simple inserts: 560s
- String concatenation: 111s
- Raw SQL: 26s
- Binary COPY: 20s

Each optimization step brought significant improvements, with our final binary implementation performing 28x faster than individual inserts while retaining all the flexibility of a high-level programming language for generating complex data patterns.

## Conclusion

Our journey from simple inserts to binary COPY protocol shows that while readable solutions are great, sometimes you need to dive into lower-level details for maximum performance. The final binary implementation may look complex, but the 28x speedup makes it worthwhile when testing databases at scale.

All code from this article is available in [repository link](<https://github.com/ignat-z/fast_generation>).
