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

# Route 53

- DNS = Domain Name System
- Common terminology:
  - Domain registrar: Route 53, Go Daddy, etc.
  - DNS records: A, AAAA, CNAME, NS, etc.
  - Zone file: contains DNS records
  - Name server: resolves DNS queries (authoritative or non-authoritative)
    - Authoritative mean you can edit the DNS records
  - Top level domain (TLD): `.com`, `.ca`, `.gov`, etc.
  - Second level domain (SLD): `amazon.com`, `google.com`, etc.
- Route 53
  - Highly available, scalable, fully managed and **authoritative** DNS
  - Domain registrar as well
  - 100% availability in SLA
- Records
  - Each record contains:
    - Hostname
    - Record type
      - A: IPv4
      - AAAA: IPv6
      - CNAME: map hostname to another hostname
        - Only works for non-root domain
      - NS: name servers for the hosted zone
    - Value
    - Routing policy: define how Route 53 responds to DNS queries
    - TTL (Time to Live)
      - Mandatory for all records, **except Alias records**
      - High TTL = less traffic on Route 53, but records may be outdated
      - Low TTL = more Route 53 traffic, records are less outdated, easy to change records
- Hosted zones
  - **Cost**: $0.50 per month per hosted zone
  - Container for DNS records
  - Defines how to route traffic to a domain and its subdomains
  - **Public hosted zones** contain records that specify how to route traffic on the Internet (public domain names)
  - **Private hosted zones** contain records that specify how you route traffic within one or more VPCs (private domain names)
- Alias records
  - Map hostname to an AWS resource (eg. ELB hostname)
  - Always an A/AAAA record (there is a toggle for _Alias_)
  - Works for root and non-root domains
  - Targets include: ELBs, CloudFront distributions, API gateway, Elastic Beanstalk environments, etc.
  - **Cannot set an Alias record for EC2 DNS name**
- Health checks in Route 53
  - Only for public resources
  - Can be used for automated DNS failover
  - What can be monitored?
    - Endpoint (eg. application, server)
    - Other health checks (calculated health check)
    - State of CloudWatch alarm
      - **Use case: monitor a private AWS resource**
        - Health checks can't work on private EC2 instance
        - Setup a CloudWatch metric on the private instance
        - Setup a CloudWatch alarm on the metric
        - Create a health check to monitor the CloudWatch alarm
- Routing policies
  - Simple
    - Route traffic to a single resource
    - Can specify multiple values in the same record; **random one chosen by the client**
    - When using Alias, specify only one resource
    - Can’t be associated with Health Checks
  - Weighted
    - **Use cases: load balancing between regions, testing new application versions, etc.**
    - Control % of request that go to each specific resource
    - You make multiple records with same name and type, set RP to weighted, and enter weight
    - Can be associated with health checks
    - Tips:
      - **Assign weight 0 to a record to stop sending it traffic**
      - **If all records have weight 0, then all records will be returned equally**
  - Latency-based
    - Redirect to the resource that has the least latency
    - **Latency is based on traffic between users and AWS regions**
    - Can be associated with health checks
  - Failover (Active-Passive)
    - Create primary and secondary (DR) resource
    - Setup health check on primary resource
    - If health check on primary resource fails, failover to the secondary resource
  - Geolocation
    - Different from latency-based; **based on user location**
    - Specify location by continent, country, state
    - A "default" record exists as a catch-all
    - **Use cases: website localization, restrict content based on location, load balancing, etc.**
    - Can be associated with health checks
  - Geoproximity
    - Route traffic based on geographic location of users and resources
    - Shift more traffic to resourced based on **bias**
      - To expand (1 to 99) - more traffic to the resource
      - To shrink (-1 to -99) - less traffic
    - Resources can be AWS resources (specify AWS region) or non-AWS resources (specify latitude & longitude)
    - **You must use Route 53 Traffic Flow to use this feature**
  - IP-based routing
    - **Routing based on clients' IP addresses**
    - You provide **list of CIDR-to-endpoint mappings**
    - **Use cases: optimize performance, reduce networking costs**
  - Multi-value
    - Use when routing traffic to multiple resources
    - Can be associated with health checks; **only healthy resources are returned**
    - Like **simple routing with multiple values**, **but ONLY healthy resources are returned**
      - Remember that simple routing **does not support** health checks
    - **Not a substitute for ELB**
- 3rd party domains & Route 53
  - Domain registrar is for buying & registering a domain name
  - Normally they also provide a DNS service to manage your DNS records for the domain
  - You can use another DNS service if you want, eg. purchase GoDaddy domain and use Route 53 for DNS management
    1. Create a public hosted zone in Route 53
    2. Update NS records on GoDaddy (or 3rd party registrar) to use Route 53 name servers

# Classic Solutions Architecture Discussions

- General note: watch these videos to understand the slides
- 3-tier web app - what are the tiers? I think typically of frontend, backend, database

---

## Case Study: WhatIsTheTime.com

- Stateless web app
- No database needed
- Start small and accept downtime
- Fully scale vertically and horizontally, no downtime

### Starting simple

- Public EC2 instance running the web app
- Elastic IP address attached to the instance

### Improvement: scaling vertically

- Respond to increased traffic to the web app
- Increase compute/memory for EC2 instance by stopping the instance and upgrading to bigger instance
  - Downtime experienced during vertical scaling

### Improvement: scaling horizontally

- Respond to the site being extremely popular now
- Multiple public large instances, no elastic IP
- A record with multiple values, TTL 1hr
- Problem: we lose an instance, but users are still hitting the IP of that instance
  - Due to high TTL

### Improvement: scaling horizontally with ELB

- Private instances
- ELB with health checks
- Restricted security groups
- Users reach ELB using alias record to the ELB's DNS name
- Problem: still manually adding instances based on traffic

### Improvement: scaling horizontally with ASG

- Switch to ASG and attach to ELB as target group
- Problem: AZ goes down, the entire application goes down

### Improvement: Making our app multi-AZ

- ASG with multi-AZ; instances are spread across AZs
- ALB with multi-AZ
- Problem: we have 3 minimum instances running, when we only need 2

### Improvement: reduce capacity

- Set minimum AZs to 2, 1 instance in each AZ
- Reserved instances
- Cost savings

### What was discussed in this case study?

- Public vs private IP and EC2 instances
- Elastic IP vs Route 53 vs load balancer
- Route 53 TTL, A records, alias records
- Maintain EC2 capacity manually vs ASG
- Multi AZ to survive disasters
- ELB health checks
- Security group rules
- Reservation of capacity for cost savings when possible

## Case Study: MyClothes.com

- Stateful web application
- Allows users to buy clothes online
- 100s of users at same time
- Scale, maintain horizontal scalability and keep web app as stateless as possible
- Users should not lose their shopping card
- Users should have their details (address, etc.) in a database

## Instantiating Applications Quickly

- EC2 instances
  - Golden AMI
  - EC2 user data
  - Hybrid: use both the above
- EBS, RDS
  - Snapshot and restore

# Elastic Beanstalk

- Web server tier: for APIs, websites, etc.
- Worker tier: for long-running tasks (eg. building, compiling, etc.)

# S3

Notes:

- S3 replication (CRR & SRR) - use cases?
- S3 event notifications
- S3 byte-range fetches - use cases?
- S3 Select & S3 Glacier Select
- S3 Access Points - VPC Origin

---

_[Next up...]_
