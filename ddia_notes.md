
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
        - memtable => most recent SSTable segment 
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
  - modern cloudnative architecture
    - snowflakes
    - databricks data lakehouse 

## Evolution and encoding

- Evovability
  - rolling upgrades: staged rollout
  - backward and forward compatibility
- binary format: 
  - good
    - mostly schema-driven: useful for documentation/code generation
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