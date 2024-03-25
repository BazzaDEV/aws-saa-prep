# Databases (RDS, Aurora, ElastiCache)

## RDS

- RDS = Relational Database Service
- Managed DB service for SQL-based databases
  - Postgres, MySQL, MariaDB, Oracle, Microsoft SQL Server, IBM DB2, AWS Aurora
- RDS vs. DB on EC2
  - **+**: Lots of stuff handled by AWS (eg. OS patching, backups/restore, read replicas, disaster recovery, scaling, EBS-backed storage)
  - **-**: Can't SSH into instances
- Storage auto-scaling
  - _Useful for applications with **unpredictable workloads**_
  - Dyanmically increase DB storage based on need
  - You set **maximum storage threshold** (max DB storage limit)
- Read replicas
  - Read scalability
  - Up to 15 read replicas
  - Within AZ, cross AZ, cross region
  - Async replication; reads are eventually consistent
  - Different connection string for each read replica
  - **Use case: reporting application**
    - Production DB is taking normal load
    - You want to run a reporting application to run some analytics
    - Solution: create read replica and run analytics workload using read replica
      - Prod. db is unaffected
  - **Network cost: free for inter-region, $$$ for intra-region**
    - Free for read replicas in the same region
    - \$\$\$ for cross-region read replicas - there is a cost for data travelling between regions
- Multi AZ (disaster recovery)
  - Sync replication
  - One DNS name - automatic app failover to standby
  - Not used for scaling
  - Read replicas can be setup as multi-AZ
- Can go from single-AZ to multi-AZ
  - Zero downtime operation - just modify DB properties
  - Behind the scenes:
    - Snapshot is taken
    - New DB is restored from snapshot in a new AZ
    - Synchronization established between DBs
- RDS Custom
  - Managed Oracle and Microsoft SQL Server databases to allow for OS and DB customization
  - Customizations:
    - Configure settings
    - Install patches
    - Enable native features
    - Access underlying EC2 instance
  - Done by deactivating automation mode

## Aurora

- Overview
  - Proprietary, closed-source
  - Drivers compatible with Postgres and MySQL
  - Cloud optimized; claims 5x performance over MySQL on RDS, 3x on Postgres on RDS
  - Up to 15 read replicas, replication process is < 10ms
  - Instant failover, high availability native
  - Costs more than RDS (20% more) but is more efficient
- High availability and read scaling
  - Creates 6 copies of your data across 3 AZs
    - 4/6 copies needed for writesw
    - 3/6 copies needed for reads
    - Self-healing, peer-to-peer replication
    - Striped storage across 100s of volumes
  - One master instance for writing
  - Automated failover for master in < 30 seconds
  - Reads provided by master + 15 read replicas
  - Read replicas are auto scaled based on workload
  - Supports cross-region replication
- Aurora cluster layout
  - Writer endpoint: points to master
  - Reader endpoint: load balancer to read replicas)
  - Reading/writing to shared storage which auto-expands
- Custom endpoints
  - Define custom RR instances (perhaps larger compute) as custom endpoint
  - Use case: run analytical queries on specific RRs
- Aurora Serverless
  - **Good for infrequent, intermittent, or unpredictable workloads**
  - Automated DB instantiation and auto-scaling based on actual usage
  - Pay per second
  - No capacity planning needed
- Global Aurora
  - (Not recommended) Aurora Cross-Region Read Replicas
    - Useful for disaster recovery
    - Simple to put in place
  - (Recommended) Aurora Global Database
    - 1 primary region (read/write)
    - Up to 5 secondary regions (read-only)
      - Cross-region replication lag is < 1 second
    - Promote secondary region for DR; RTO of < 1 minute
- Aurora Machine Learning
  - Add ML-based predictions to your applications via SQL
  - Provided by AWS ML services (SageMaker, Comprehend)
  - Use cases: fraud detection, ads targeting, sentiment analysis, production recommendations
- Aurora Database Cloning
  - **Useful for creating a "staging" database from "production" without impacting the production database**
  - Create a new Aurora DB cluster from an existing one
  - Faster than snapshot & restore method
  - Uses copy-on-write protocol
    - When clone is created, no data is copied; clone references same data as source
    - Clone and source only share unaltered pages; modifications after the cloning process are exclusive to each cluster.
    - When a change is made to source data:
      - Source makes copy of the shared data
      - Source modifies the copied data
      - Source now references the modified data, while the clone still references the original data
      - This applies vice-versa

## Backup & Restore

### Backup

#### RDS

- Automated backups
  - Daily full backup of DB (during backup window)
  - Transaction logs backed up every 5 minutes
  - Restore to any point in time
  - 1 to 35 days of retention for backups (set to 0 to disable)
- Manual backups
  - DB snapshot
  - Unlimited retention period
- **Trick: If you plan to stop RDS DB for long time, snapshot & restore instead of stopping it.**

#### Aurora

- Automated backups
  - 1 to 35 days (cannot be disabled)
  - Point-in-time recovery in that timeframe
- Manual backups
  - DB snapshot
  - Unlimited retention period

### Restore

- Restoring RDS/Aurora backup or snapshot creates a new database
- Restoring on-premises DB to RDS
  - Backup on-premises DB
  - Upload to S3
  - Restore backup file onto new RDS instance running same DB (eg. Postgres, MySQL, etc.)
- Restoring on-premises DB to Aurora cluster
  - Backup on-premises DB using Percona XtraBackup
  - Upload to S3
  - Restore backup file onto new Aurora cluster running same DB (Postgres or MySQL)

## Security

- At-rest encryption
  - Master and replicas encrypted using AWS KMS - must be defined at launch time
  - If master is not encrypted, RRs cannot be encrypted
  - Encrypt an unencrypted DB using DB snapshot and restore as encrypted method
- In-flight encryption: TLS-ready by default, uses AWS root certificates
- IAM authentication (eg. IAM roles) instead of user/pass
- Security groups
- No SSH available except RDS Custom
- Audit logs; send to CloudWatch Logs for longer retention

## RDS Proxy

- **Improves DB efficiency by reducing stress on resources and minimizing open connections**
- Full managed DB proxy for RDS/Aurora
- Allows apps to pool and share DB connections
- Serverless, autoscaling, highly available (multi-AZ)
- **Reduced RDS/Aurora failover time by 66%**
- **Enforce IAM authentication for DB, store credentials in AWS Secrets Manager**
- **RDS proxy only reachable from inside VPC**
- **Use case: Lambda functions making DB calls**

## ElastiCache

- Overview
  - **Requires heavy application code changes**
  - Managed Redis or Memcached
  - Cache: in-memory DB; high performance, low latency
  - Reduces DB load for read-intensive workloads
  - Make your application stateless by storing in cache
- **Solution architecture: DB cache**
  - Application queries cache
    - Cache hit: read data from cache
    - Cache miss: read data from DB, write to cache for future use
  - Invalidation strategy required
- **Solution architecture: User session store**
  - User logs into application via Instance A
  - Application writes session data into cache
  - User hits Instance B on next use
  - Application in Instance B grabs session data for user from cache, user is logged in!
- Redis vs. Memcached
  - Redis for high availability, backup/restore, read replicas, multi-AZ with auto-failover
  - Memcached for data you can lose
- Cache security
  - IAM auth for Redis
  - Redis local auth
    - Password/token when creating Redis cluster
    - Use as an extra layer of security, on top of security groups
    - Supports SSL for in-flight encryption
  - Memcached supports SASL-based auth (advanced, not covered here)
- ElastiCache patterns
  - **Lazy loading:** all read data is cached, data can be stale in cache
  - **Write through:** update cache data when writing to DB; no stale data
  - **Session store:** store temporary session data in cache
- **ElastiCache use case: gaming leaderboard**
  - Red sorted sets guarantee uniqueness and element ordering
  - Each time a new element is added, it is ranked in real-time, then added in correct order

