# Finacle HTAP Implementation: Dual Storage Under the Hood

## Real-World Scenario: Multi-Branch Banking Operations

Consider a Finacle-powered bank with 500 branches processing thousands of daily transactions while simultaneously generating real-time risk reports, regulatory compliance dashboards, and customer analytics.

## Database Schema Setup

```sql
-- Core banking transaction table
CREATE TABLE account_transactions (
    txn_id NUMBER PRIMARY KEY,
    account_id NUMBER NOT NULL,
    customer_id NUMBER NOT NULL,
    amount NUMBER(15,2) NOT NULL,
    txn_type VARCHAR2(20) NOT NULL,
    branch_id NUMBER NOT NULL,
    currency_code CHAR(3) DEFAULT 'USD',
    txn_timestamp TIMESTAMP DEFAULT SYSTIMESTAMP,
    balance_after NUMBER(15,2),
    teller_id NUMBER,
    channel VARCHAR2(10) -- ATM, BRANCH, MOBILE, ONLINE
);

-- Enable dual storage: Row + Columnar
ALTER TABLE account_transactions INMEMORY PRIORITY HIGH;

-- Optimize for common OLTP access patterns
CREATE INDEX idx_account_txn ON account_transactions(account_id, txn_timestamp);
CREATE INDEX idx_customer_txn ON account_transactions(customer_id, txn_timestamp);
```

## OLTP Operations: Real-Time Transaction Processing

### Customer Deposit Transaction

**Finacle Application Flow:**
```sql
-- Customer deposits $5,000 at Branch 101
BEGIN
    -- Insert transaction record
    INSERT INTO account_transactions (
        txn_id, account_id, customer_id, amount, txn_type, 
        branch_id, balance_after, teller_id, channel
    ) VALUES (
        txn_seq.NEXTVAL, 12345, 8901, 5000.00, 'DEPOSIT',
        101, 25000.00, 204, 'BRANCH'
    );
    
    -- Update account balance (separate table)
    UPDATE accounts 
    SET current_balance = current_balance + 5000.00,
        last_txn_date = SYSTIMESTAMP
    WHERE account_id = 12345;
    
    COMMIT;
END;
```

**Under the Hood - Row Storage:**
```
Physical Row Layout (Heap Table):
[txn_id: 501001][account_id: 12345][customer_id: 8901][amount: 5000.00]
[txn_type: 'DEPOSIT'][branch_id: 101][currency: 'USD'][timestamp: 2025-09-13 14:30:15]
[balance_after: 25000.00][teller_id: 204][channel: 'BRANCH']
```

**OLTP Query - Account Balance Inquiry:**
```sql
-- Customer checks balance via mobile app
SELECT balance_after 
FROM account_transactions 
WHERE account_id = 12345 
ORDER BY txn_timestamp DESC 
FETCH FIRST 1 ROW ONLY;

-- Oracle uses ROW storage via index
-- Execution: Index seek → Single row fetch → Return result in 2ms
```

### Concurrent Money Transfer

**While deposit is processing, simultaneous transfer occurs:**
```sql
-- Transfer $1,200 from Account 67890 to Account 54321
BEGIN
    -- Debit transaction
    INSERT INTO account_transactions VALUES (
        txn_seq.NEXTVAL, 67890, 7654, -1200.00, 'TRANSFER_OUT',
        102, 8300.00, 205, 'ONLINE'
    );
    
    -- Credit transaction  
    INSERT INTO account_transactions VALUES (
        txn_seq.NEXTVAL, 54321, 3456, 1200.00, 'TRANSFER_IN',
        102, 4700.00, 205, 'ONLINE'
    );
    
    COMMIT;
END;
```

## Columnar Storage Synchronization

**Asynchronous Columnar Update (happens within 100-500ms):**
```
Column-Oriented Physical Layout:

TXN_ID Column:     [501001, 501002, 501003, ...]
ACCOUNT_ID Column: [12345,  67890,  54321,  ...]  
AMOUNT Column:     [5000.00, -1200.00, 1200.00, ...]
TXN_TYPE Column:   ['DEPOSIT', 'TRANSFER_OUT', 'TRANSFER_IN', ...]
BRANCH_ID Column:  [101, 102, 102, ...]
TIMESTAMP Column:  [14:30:15, 14:30:18, 14:30:18, ...]
```

**Compression Applied:**
- **Dictionary Encoding**: TXN_TYPE → {1:'DEPOSIT', 2:'TRANSFER_OUT', 3:'TRANSFER_IN'}
- **Run-Length Encoding**: Consecutive same branch_ids compressed
- **Delta Compression**: Timestamps stored as deltas from base value

## OLAP Operations: Real-Time Analytics

### Risk Management Dashboard

**Real-Time Fraud Detection Query:**
```sql
-- Detect accounts with unusual activity patterns
SELECT 
    account_id,
    COUNT(*) as txn_count,
    SUM(CASE WHEN amount > 0 THEN amount ELSE 0 END) as total_deposits,
    SUM(CASE WHEN amount < 0 THEN ABS(amount) ELSE 0 END) as total_withdrawals
FROM account_transactions 
WHERE txn_timestamp >= SYSTIMESTAMP - INTERVAL '1' HOUR
    AND ABS(amount) > 10000
GROUP BY account_id
HAVING COUNT(*) > 5
ORDER BY txn_count DESC;
```

**Under the Hood - Columnar Processing:**
```
Oracle automatically uses In-Memory Column Store:

1. Scan TIMESTAMP column → Filter last hour (vectorized operation)
2. Scan AMOUNT column → Filter >10000 (SIMD instructions)
3. Scan ACCOUNT_ID column → Group aggregation
4. Parallel execution across multiple CPU cores
5. Result: Sub-second response for millions of records
```

### Regulatory Compliance Report

**Basel III Capital Adequacy Calculation:**
```sql
-- Real-time exposure calculation across all branches
SELECT 
    b.region,
    b.branch_name,
    SUM(CASE WHEN t.amount > 0 THEN t.amount ELSE 0 END) as total_deposits,
    COUNT(DISTINCT t.customer_id) as active_customers,
    AVG(t.amount) as avg_transaction_size,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY ABS(t.amount)) as p95_amount
FROM account_transactions t
JOIN branches b ON t.branch_id = b.branch_id
WHERE t.txn_timestamp >= TRUNC(SYSDATE) -- Today's transactions
GROUP BY b.region, b.branch_name
ORDER BY total_deposits DESC;
```

**Columnar Advantage:**
- **I/O Efficiency**: Reads only required columns (amount, customer_id, branch_id, timestamp)
- **Compression**: 10x less data movement compared to row storage
- **Vectorization**: SIMD instructions process multiple values simultaneously
- **Parallel Execution**: Automatic parallelization across CPU cores

## Resource Isolation and Performance

### Memory Pool Allocation
```sql
-- Configure resource allocation
ALTER SYSTEM SET inmemory_size = 8G;           -- Columnar storage
ALTER SYSTEM SET sga_target = 16G;             -- Row storage + other
ALTER SYSTEM SET parallel_max_servers = 32;    -- OLAP query parallelism
```

### Workload Management
```sql
-- Create resource plan for workload isolation
BEGIN
    DBMS_RESOURCE_MANAGER.CREATE_PLAN(
        plan => 'FINACLE_WORKLOAD_PLAN'
    );
    
    -- OLTP gets priority
    DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
        plan => 'FINACLE_WORKLOAD_PLAN',
        group_or_subplan => 'OLTP_GROUP',
        cpu_p1 => 70,              -- 70% CPU priority
        parallel_degree_limit_p1 => 2  -- Limited parallelism
    );
    
    -- OLAP gets remaining resources
    DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
        plan => 'FINACLE_WORKLOAD_PLAN',
        group_or_subplan => 'OLAP_GROUP',
        cpu_p1 => 30,              -- 30% CPU when OLTP is busy
        parallel_degree_limit_p1 => 16 -- Higher parallelism allowed
    );
END;
```

## Performance Characteristics

### OLTP Performance (Row Storage)
```
Transaction Type          | Response Time | Throughput
Customer Deposit         | 3-5ms         | 50,000 TPS
Balance Inquiry          | 1-2ms         | 100,000 TPS  
Money Transfer           | 8-12ms        | 25,000 TPS
Account Statement        | 15-25ms       | 10,000 TPS
```

### OLAP Performance (Columnar Storage)
```
Query Type                    | Data Scanned | Response Time
Hourly transaction summary    | 1M records   | 200ms
Daily branch performance      | 10M records  | 1.2s
Monthly customer analytics    | 100M records | 8s
Quarterly risk assessment     | 500M records | 45s
```

## Business Value Realization

**Operational Benefits:**
- **Sub-second fraud detection** on live transaction streams
- **Real-time regulatory reporting** without batch processing delays
- **Immediate customer insight** for personalized banking services
- **Zero impact** on core banking transaction performance

**Technical Benefits:**
- **Single data source** eliminates ETL complexity and data consistency issues
- **Automatic query optimization** routes workloads to optimal storage format
- **Resource isolation** prevents analytical queries from affecting customer transactions
- **Cost efficiency** through storage compression and elimination of separate analytical systems

## Monitoring and Observability

```sql
-- Monitor columnar storage effectiveness
SELECT 
    table_name,
    inmemory_size,
    inmemory_compression,
    populate_status,
    priority
FROM v$im_segments 
WHERE table_name = 'ACCOUNT_TRANSACTIONS';

-- Track query execution paths
SELECT 
    sql_text,
    executions,
    avg_etime,
    buffer_gets,
    disk_reads,
    inmemory_io_cell_offload_returned_bytes
FROM v$sql 
WHERE sql_text LIKE '%account_transactions%'
ORDER BY executions DESC;
```

This implementation demonstrates how Finacle leverages unified HTAP architecture to deliver both millisecond transaction processing and real-time business intelligence from the same underlying data, without compromising either workload's performance requirements.
