# Measuring transaction performance: MongoDB meets TPC-C
**Speaker: asya kamsky @asya999**

How did we benchmark transaction performance?

motivation:
- 'benchmarketing' (finding niche benchmarking numbers for marketing) is useless
- benchmarks in the world assume SQL
    + frequently badly executed
- after introducing transactions (Mongo 4.0), needed to know how fast they were, how they're doing...

Made use of "evergreen", mongo's continuous testing platform. runs benchmarks. alerts when things are bad. helps us see regressions, expected or unexpected.

Methodology:
- what is good workload to benchmark transactions?
- "on the state of nosql benchmarks". not really anything. ycsv is closest. but not really meant for complex data, just key value stores.

The TPC-C Benchmark is a transaction processing benchmark:
- 1988: transaction processing performance council. analysis for data queries.
- tpc benchmark c, ~132 pages.
- data model: warehouse. 1 company with 1 or more warehouses. # warehouses determines scale. each warehouse has 10 districts, each district has 3000 customers, some orders, items.
    + workload: transactions: new order, delivery (10 orders), payment, order status check (read only), stock level check (read only)
- data schema: relational
- 44% new orders, 44% payments.... new orders processed per minute

Looked to existing implementations of TPC-C:
- `py-tpcc`
    - implements TPC-C for a bunch of nosql databases
    - written around 2011

- decided to work off of `py-tpcc` and modify it for our use case.

Evolution of tpc-c for mongoDB
-  have to write tpc transaction in database transaction (didn't have transactions in nosql at the time). wrap with mongo 4.0 db transaction
- connect to replica set
- have a/b setting (run with and without transactions)
- schema
    + original py-tpcc: two modes -- normalized vs denormalized
        * original py-tpcc embedded too many things in customer.
        * schema design: order lines should be embedded in orders. since order lines are limited. once order is done, you're done with order lines.
            - indexes shrink
        * customer data for example, keeps growing. if all amazon orders embedded in customer it would exceed limit of a single document.
    + denomralized better throughout.
- fix indexes
    + suboptimal indices. with any field that would be queried, created single field query. most should be combined. custoemr index only relevant in specific district, for example.
- hardware selection
    - Atlas, MongoDB DaaS: replica set (3 nodes), gce, m80 (also m50 & m60), us-east1
        + client: GCE, n1-highcpu-16 (same region--also us-east1)
    + no hw/os configuration decisions
    + 4.0.latest
    + automatically updated/patched/etc
    + monitoring
- tpc-c cost per "T" in TpmC 
    + comapre cost of m80, m50, m60
- code optimization
    + least amount of retry / time to rollback, etc. ex. reads don't take lock but writes do. order of reads and writes.
    + findAndModify. update and return. returns entire document after update.
        * single write instead of a read and then a write. (was reading for document and then updating it). 
    - reduce round trips to db
- secondary reads
    + can help sometimes
        + if closer than primary, faster b/c of latency of roundtrip.
        + ETL or analytical queries will mess up working set / trash your cache. dedicated read-only secondaries. 
        + transactions can't write on secondary. only order/stock status could go to secondaries. they were in same region. reads were less than 2% of operations. offloading them to secondaries didnt' have measurable advantage.
    + secondary are doing all the writes primary is doing but doesn't have to worry about rollbacks. 
- write concern
    + default is 1
    + used w=majority
- code improvement for new order
- batching deliveries
    +  multiple transactions with w=majority vs w=1. waits for majority for every transaction vs a single one.
- measure everything
- used mongodb charts

- results: m50 < m60 < m80 but not worth based on cost. performance improvement was less than proportional cost difference.
- better to scale out rather than up
