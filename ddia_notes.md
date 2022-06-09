
# Ch 1. Foundations of Data Systems


## how to define a good system

- Reliability: continue to work even when faults occur. 
- Scalability: continue to work even when load increases.
- Maintainability: 
  - good abstraction: adapt to new requirements easily
  - good operability: observability, testable

## data model

- data model
  - relational      
    - good
      - better support for join
    - bad
      - object relational mismatch
      - bad design, unnormalized tables => write overheads, inconsistency risks
  - NoSQL document     
    - good
      - flexible schema enforced on read/write
      - better performance because of locality, more scalable
      - no object-relational mismatch, closer to data structure in application 
      - good for one-to-many TREE-structured data
    - bad
      - weak support for join, normally emulated in application code
  - NoSQL graph
    - good for many to many relationship

## storage and retrieval

- OLTP user-facing, indexed, bottle-necked by disk seek time
  - Storage engine
    - B+Tree       update in-place -- treat disk as fixed sized pages that can be overwritten
      - ordered balanced search tree, good support for range search
    - LSM tree:    append only, delete old files with compaction
      - Write:
        - **batch writes in in-memory balanced tree memtable** 
          - `memtable ` size > threshold: write to `SSTable ` on disk, new SStable file becomes most recent segment 
      - Read
        - read memtable first as cacheing layer => most recent SSTable segment on disk
        - Bloomfilter to improve false-negative read. (if not there, then definitely not there)
          - apply a group of hash functions on input, mark output on a bit array
      - **Durability and crash recovery**: **WAL**
        - Overtime, run a merge and compaction process in background to combine segments and drop overwritten/deleted value
      - Use case
        - Bigtable -> Hbase random access OLTP distributed KV store
        - Bigtable -> LevelDB -> RocksDB (Single Machine embeded kv storage engine)
          - Cassandra, Kafka Stream, CockrouchDB, MySQL-MyRocks
          - TiDB - TiKV
- OLAP
  - traditional data warehousing MPP query engine + column store
    - https://www.devopsschool.com/blog/what-is-apache-kudu/
- modern cloudnative architecture: seperation of computation and storage
  - "htap" tidb
  - snowflakes
  - databricks data lakehouse 

## Evolution and encoding

- Evovability
  - rolling upgrades: staged rollout / canary release
  - backward and forward compatibility
- binary format: 
  - good
    - mostly schema-driven: useful for documentation/code generation
      - reduce risks on deploying new versions
      - rolling incrementally by portions / nodes
    - clear semantics on backward/forward compatibility
    - good performance
  - bad
    - not human readable
  - examples
    - protocol buffers
    - thrift
    - archival storage on hadoop
      - avro, parquet (columnar)
- textual format
  - more widely used
  - example
    - json, csv, yaml, xml
- modes of dataflow / cross-service communication
  - Database
    - one service writes encoded data, another service read it sometime in the future, or read by itself in the future
  - HTTP
    - good for experimentation, debugging, good ecosystem
    - usually for Publics APIs
  - RPC
    - better performance with specialized binary encoding format
    - usually for Internal APIs
  - Async Message Passing 
    - low latency like RPC + durability like database

# Ch 2. Distributed Data
- why replicated data
  - geographically close to user, reduce latency
  - fault tolerance, increase availbility
  - scale out (horizontally) to increase read throughput
- difficulties in replication
  - handling **changes** in replicated data
    - single-leader 
    - multi-leader
    - leaderless
  - trade-offs, usually configuration options
    - synchronous vs asynchronous
    - replica failure ?
## Leaders and followers
- replica: a node that stores a copy of the database
  - how to ensure all the data ends up on all replicas?
    - every write should be processed by every replica, otherwise data insonsistency
![](https://raw.githubusercontent.com/th2zz/typora_img_host/master/img/Screenshot%202022-06-09%20094738.png)
- leader-based replication / active-passive / master-slave
  - clients writes to leader => leader writes to local storage
  - leader sends data change to all followers as part of a **replication log** / change stream
  - each follower (read replica, slave, secondary, hot standby) takes the log from the leader
      - updates local copy in the **same order** as leader
  - examples
      - relational databases
          - PostgreSQL 9.0+
          - MySQL
          - SQL Server's AlwaysOn availbility group
      - nonrelational 
          - MongoDB
          - RethinkDB
          - Espresso
      - Distributed message brokers
          - Kafka
          - RabbitMQ HA queues
      - Some network file systems and replicated block devices
          - DRBD
## Synchronous vs Asynchronous Replication
- synchronous replication
  - replica has guranteed **update-to-date consistent copy** of the data
  - will **block write requests** until replica available again
    - **impractical** to make all followers synchronous: any one-node outage => halt
- **semi-synchronous** usually do 1 sync follower + the others async standby followers to replace the sync follower 
- **mainstream** in leader-based replication **fully asynchronous** set-up: 
    - write is not guranteed to be durable: a failed leader can lost all updates that are not replicated
    - nonblocking writes: leader not blocked by followers' progress
    - commonly used in many-followers / geographically distributed systems
## Setting up new followers
- naive bootstrapping process does not make sense:
  - simply copy data files: data is in flux, writing may hit leader, different parts may have different outlook
  - lock database for writes? decreases availbility
- zero downtime new followers bootstrap (some system automated, others requires manual operations)
  - take a snapshot of leader's database without locking it 
    - commonly supported for backups
  - copy snapshot to new follower 
  - follower automatically requests all data changes that have happened since the snapshot was taken.
    - snapshot is associated with an exact position in leader's replication log
      - PostgreSQL: log sequence number
      - MySQL: binlog coordinates
  - follower gradually caught up and in sync

## Handling Node outages
- how to keep system running with limited impact from node down-time
- How to achieve high availbility with leader-based replication ?
- Follower failure
  - catch up recovery
    - follower recovery from its own WAL log, requests data changes from leader
- Leader failure
  - failover
    - one follower becomes new leader, clients need reconfiguration, other followers now consumes data from new leader
    - general automatic failover process
      - determine leader dead by heartbeat / healthcheck
      - leader election, best candidate usually the replica with the most up-todate data
      - reconfigure the system entirely to use new leader: clients and all other followers
        - old leader comes back => it should recognize new leader as well
    - common issues
      - conflicting writes in async replication set-up
        - new leader may have hot received all writes from failed old leader
        - most common solution: discard unreplicated writes => violates clients durability expectations
          - discarding writes is dangerous especially if external coordination is involved
            - https://github.blog/2012-09-14-github-availability-this-week/
      - split brain: two nodes both believes they are leader
        - if both accepting writes => no process for resolving conflicts => data lost/corruption
        - solution
          - if automatic detection mechanism is used, it should be carefully designed
            - bad detection can lead to both nodes shut down
            - https://github.blog/2012-12-26-downtime-last-saturday/
          - some ops team prefer manual failover
      - inapproperiate healthcheck timeout for dead leader detection
        - too long => longer recovery time
        - too short => unnecessary failovers
          - load spike / network glitch => increased response time 
          - an uncessary failover can negatively affect system under high load
## Leader-based replication Impelentations
- statement-based replication logging
  - process
    - leader logs every writes request (statement) it executes
      - e.g. Relational: INSERT, UPDATE, DELETE
    - sends log to followers
    - each follower parses & executes the SQL statement
  - Issues
    - nondeterministic function calls can have different value on each replica: NOW(), RAND()
    - p.159 181

# Ch 3. Derived Data

- Batch processing:
  - e.g. Hive: a SQL interface on top of MapReduce + HDFS
  - Key assumption: input is bounded 
- Streaming processing:
  - input unbounded
  - more real-time and performant

- Terminology 
  - Producer, Consumers
  - Topic: a group of related events
    - similar analogy:  filename identifies a set of related records
- 