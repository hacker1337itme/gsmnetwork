# gsmnetwork
gsmnetwork

## HLR Database Technologies

### 1. **Traditional Telco-Grade Databases**

Historically, HLRs were built on specialized, proprietary database systems optimized for telecommunications:

#### **In-Memory Databases**
- **Why:** HLR lookups need to happen in milliseconds (typically 10-50ms)
- **Examples:**
    - **Oracle TimesTen** - In-memory database widely used in telecom
    - **Altibase** - Popular in Asian telecom markets
    - **SolidDB** - IBM's in-memory database used in some HLRs
    - **McObject eXtremeDB** - Real-time embedded database

#### **High-Performance Relational Databases**
- **Why:** Need ACID compliance for subscriber data integrity
- **Examples:**
    - **Oracle RAC (Real Application Clusters)** - With specific telecom optimizations
    - **IBM DB2** - Especially in older Ericsson HLRs
    - **Sybase ASE** - Used in some Nokia HLR implementations

### 2. **Proprietary HLR Implementations**

Major telecom equipment vendors built their own database layers:

#### **Ericsson HLR**
- Uses **Ericsson APZ** processor with **PLEX** programming language
- Database: **Ericsson DBS (Database System)** - proprietary hierarchical database
- Data stored in **CP (Central Processor)** memory for speed

#### **Nokia HLR**
- Originally used **TID (Telecommunications Information Database)** 
- Later moved to **Oracle**-based solutions
- **Nokia mwINSIGHT** platform

#### **Huawei HLR**
- **iHLR** product uses proprietary database optimized for telecom
- Based on **Huawei VRP (Versatile Routing Platform)** 
- Can use **Oracle** or **Huawei GaussDB** in modern deployments

#### **ZTE HLR**
- **ZXWN HLR** uses proprietary database layer
- Can be configured with **Oracle** or **MySQL Cluster**

### 3. **Database Types Used**

| Database Type | Characteristics | Used For |
|--------------|-----------------|----------|
| **Hierarchical** | Tree-like structure, fast lookups | Original GSM HLRs (Nokia, Ericsson) |
| **Relational** | Tables, ACID compliance, complex queries | Modern HLRs, subscriber profiles |
| **Key-Value Store** | Simple, extremely fast lookups | Subscriber location data |
| **In-Memory** | Microsecond latency, volatile | Real-time lookups, active subscriber set |

### 4. **Modern HLR/HSS Implementations**

With 4G/5G evolution, HLR became HSS, and now uses more modern technologies:

#### **Cloud-Native HSS**
- **Cassandra** - Used by some telcos for distributed subscriber data
- **Redis** - For caching frequently accessed subscriber profiles
- **MongoDB** - Document store for flexible subscriber attributes
- **Aerospike** - Flash-optimized, used for large-scale subscriber management

#### **Examples:**
- **AWS for Telecom** - Some operators run HLR/HSS on AWS using DynamoDB
- **Google Cloud Telecom** - Using Spanner for subscriber databases
- **Microsoft Azure for Operators** - Deploying HSS on Cosmos DB

### 5. **Data Volume Example**

A typical HLR for a mid-sized operator (10 million subscribers):

```
Data per subscriber: ~2-5 KB
Total active data: 20-50 GB in memory
Backup/archive: Several terabytes
Transactions: 1000-5000 lookups/second
Peak load: Up to 10,000 TPS
```

### 6. **Real-World Implementation Example**

Let's look at how data is actually stored in an HLR:

```sql
-- Simplified HLR table structure (if relational)
CREATE TABLE SUBSCRIBER (
    IMSI VARCHAR(15) PRIMARY KEY,        -- International Mobile Subscriber Identity
    MSISDN VARCHAR(15),                    -- Phone number
    KI VARCHAR(32),                        -- Authentication key (encrypted)
    SUBSCRIBER_STATE VARCHAR(10),           -- Active/suspended/barred
    BASIC_SERVICES BITMAP(64),              -- Services subscribed to
    CREATED_DATE TIMESTAMP,
    MODIFIED_DATE TIMESTAMP
);

CREATE TABLE LOCATION_INFO (
    IMSI VARCHAR(15) PRIMARY KEY,
    CURRENT_VLR VARCHAR(15),                -- Which VLR serves this subscriber
    CURRENT_MSC VARCHAR(15),                 -- Current MSC
    LOCATION_AREA VARCHAR(10),               -- Current location area
    LAST_UPDATE TIMESTAMP,
    FOREIGN KEY (IMSI) REFERENCES SUBSCRIBER(IMSI)
);

CREATE INDEX idx_msisdn ON SUBSCRIBER(MSISDN);
```

But in reality, HLRs use highly optimized data structures:

```c
// Pseudo-code for HLR in-memory structure
struct hlr_subscriber {
    uint64_t imsi;                    // 64-bit ID for fast lookup
    char msisdn[15];                   // Phone number
    uint8_t ki[16];                    // 128-bit authentication key
    uint32_t services_bitmask;          // 32-bit for 32 services
    uint16_t current_vlr_id;            // Index to VLR table
    uint16_t location_area;              // Location area code
    time_t last_update;                  // Timestamp
    // ... other fields
};

// Hash table for O(1) lookup by IMSI
struct hlr_subscriber* subscriber_table[1 << 24];  // 16 million entries
```

### 7. **Key Technical Requirements**

- **99.999% availability** (< 5 minutes downtime per year)
- **< 10ms response time** for 95% of queries
- **Synchronous replication** to backup site (usually within 50ms)
- **Geographic redundancy** (active-active or active-standby)
- **Transactional integrity** (no lost updates, even during failures)

### 8. **Database Operations in HLR**

The HLR database must support these key operations efficiently:

1. **UpdateLocation** - Update subscriber's current VLR
2. **SendAuthenticationInfo** - Retrieve authentication vectors
3. **InsertSubscriberData** - Send profile to VLR
4. **DeleteSubscriberData** - Remove subscriber
5. **AnyTimeInterrogation** - Query subscriber location/state

Each operation typically requires:
- 1-3 database lookups
- Sub-10ms completion time
- Transaction logging for recovery

