# Twitter Workload (BenchBase) — Stress Test for High Connection Concurrency

A high-concurrency mixed OLTP benchmark that models typical social-network operations and heavily stresses connection handling, contention, and latency under thousands of parallel sessions.

## Test setup
**Platforms:** OceanBase (3-node cluster), PostgreSQL 17 (synchronous cluster), Oracle 19c (ADG), SQL Server 2022 (standalone)  
**Hardware (all servers):** 16 vCPU, 128 GB RAM, SAN storage

- **Scale factor:** 80,000  
- **Terminals:** 4,000  
- **Duration:** 60 minutes  
- **Isolation:** READ COMMITTED  
- **Final State:** EXIT  

**BenchBase changes:** HikariCP was used for all DBMS runs. In addition, the OceanBase DDL scripts were modified to introduce partitioning for load distribution, and the official OceanBase JDBC driver was used. Other DBMS runs were left unchanged.

## Short result
OceanBase demonstrates not only efficient handling of **4,000 sessions** (similar to SQL Server) thanks to a modern **thread-based connection processing model**, but also the ability to **distribute load across cluster nodes**, making it the clear leader for **high-concurrency mixed OLTP workloads**.

## Summary table
| Metric | OceanBase CE | SQL Server | PostgreSQL 17 | Oracle |
|---|---:|---:|---:|---:|
| Measured Requests | 205,703,899 | 132,737,203 | 34,613,846 | 32,700,991 |
| Throughput (rps) | 57,124.11 | 36,861.21 | 9,612.29 | 9,081.09 |
| Goodput (rps) | 57,127.25 | 36,871.35 | 9,615.58 | 9,084.95 |
| Min Latency | 0.434 ms | 0.384 ms | 0.153 ms | 0.128 ms |
| Median Latency | 6.536 ms | 103.958 ms | 361.926 ms | 372.543 ms |
| p90 Latency | 220.1 ms | 133.5 ms | 852.8 ms | 954.6 ms |
| p95 Latency | 262.6 ms | 145.7 ms | 1,025.0 ms | 1,160.1 ms |
| p99 Latency | 363.2 ms | 168.9 ms | 1,401.5 ms | 1,595.1 ms |
| Max Latency | 124,141.8 ms | 21,537.0 ms | 41,805.8 ms | 4,642.0 ms |
| Average Latency | 70.01 ms | 108.47 ms | 416.05 ms | 440.38 ms |

## Key takeaways
🟩 **1) OceanBase leads in throughput**  
- OB: **57,124 rps**  
- SQL Server: **36,861 rps** → **1.55× lower** than OB  
- PostgreSQL: **9,612 rps** → **5.94× lower** than OB  
- Oracle: **9,081 rps** → **6.29× lower** than OB  

🟩 **2) OceanBase completes the most requests per hour**  
- OB: **205.7M**  
- SQL Server: **132.7M** (≈ **1.55× fewer** than OB)  
- PostgreSQL: **34.6M** (≈ **5.94× fewer**)  
- Oracle: **32.7M** (≈ **6.29× fewer**)  

🟩 **3) Latency: OceanBase has the best p50, SQL Server has the best tails**  
- **Median (p50):** OB = **6.5 ms** (best); SQL Server = **104 ms**; PG/Oracle ≈ **362–373 ms**  
- **p95/p99:** SQL Server is best (**p95 ≈ 146 ms, p99 ≈ 169 ms**); OB tails are higher (≈ **263/363 ms**)  

🟧 **4) Minimum latency is better on Oracle/PG, but it does not drive throughput**  
- Min: Oracle **0.128 ms**, PG **0.153 ms**, SQL **0.384 ms**, OB **0.434 ms** — occasional ultra-fast responses don’t define behavior under **4,000 terminals**.  

🟥 **5) Max latency**  
- OB max ≈ **124 s** — rare outliers; the others are much lower (SQL ≈ **21.5 s**, PG ≈ **41.8 s**, Oracle ≈ **4.6 s**).  
