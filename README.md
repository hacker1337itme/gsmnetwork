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

I understand you're asking about the top 20 attacks on GSM network databases (HLR, VLR, AuC, EIR). This is a critical security topic. Let me provide a comprehensive list of the most significant attacks against these telecom databases.

## Top 20 Attacks on GSM Network Databases

### **SS7 Protocol-Level Attacks** (Most Common)

#### 1. **HLR Lookup Attack**
- **Target:** HLR database
- **Method:** Attacker sends `sendRoutingInfo` MAP message to HLR
- **Gains:** Subscriber's IMSI, current VLR, MSC address
- **Impact:** Location tracking, call interception setup

#### 2. **Location Tracking via AnyTimeInterrogation**
- **Target:** HLR/VLR
- **Method:** Sends `AnyTimeInterrogation` (ATI) MAP message
- **Gains:** Current cell ID, location area, subscriber state
- **Impact:** Real-time physical location tracking of target

#### 3. **Subscriber Data Modification**
- **Target:** HLR
- **Method:** `insertSubscriberData` MAP message spoofing
- **Gains:** Modify call forwarding, service settings
- **Impact:** Redirect calls/SMS, deny service

#### 4. **Cancel Location Attack**
- **Target:** HLR/VLR
- **Method:** `cancelLocation` MAP message spoofing
- **Gains:** Deregister subscriber from network
- **Impact:** Denial of service, force phone to re-register (expose location)

#### 5. **PurgeMS Attack**
- **Target:** HLR
- **Method:** `purgeMS` MAP message
- **Gains:** Remove subscriber from HLR and VLR
- **Impact:** Complete service denial, require restart to recover

#### 6. **Authentication Vector Theft**
- **Target:** AuC/HLR
- **Method:** `sendAuthenticationInfo` MAP abuse
- **Gains:** Authentication triplets (RAND, SRES, Kc)
- **Impact:** Clone SIM, intercept calls, impersonate subscriber

#### 7. **ProvideRoamingNumber Attack**
- **Target:** VLR/HLR
- **Method:** `provideRoamingNumber` MAP message
- **Gains:** MSRN (temporary roaming number)
- **Impact:** Route calls to attacker, intercept incoming calls

#### 8. **RestoreData Attack**
- **Target:** HLR
- **Method:** `restoreData` MAP message
- **Gains:** Complete subscriber profile
- **Impact:** Massive data breach of subscriber information

#### 9. **UpdateLocation Spoofing**
- **Target:** HLR
- **Method:** `updateLocation` from fake VLR
- **Gains:** Redirect subscriber location to attacker-controlled VLR
- **Impact:** Full call/SMS interception

#### 10. **RegisterSS/EraseSS Attack**
- **Target:** HLR
- **Method:** `registerSS` or `eraseSS` supplementary service manipulation
- **Gains:** Modify call forwarding, waiting, barring settings
- **Impact:** Service manipulation, call interception

---

### **Database Direct Attacks**

#### 11. **SQL Injection on HLR/HSS**
- **Target:** HLR database layer (if using SQL-based system)
- **Method:** Exploit OAM (Operations & Maintenance) interfaces
- **Gains:** Extract all subscriber data, modify records
- **Impact:** Complete database compromise

#### 12. **NoSQL Injection**
- **Target:** Modern HSS (MongoDB, Cassandra based)
- **Method:** Inject commands via unvalidated inputs
- **Gains:** Bypass authentication, extract data
- **Impact:** Data breach of millions of subscribers

#### 13. **Privilege Escalation via OAM Interface**
- **Target:** HLR management interface
- **Method:** Exploit weak access controls on admin panels
- **Gains:** Full database admin access
- **Impact:** Complete network compromise

#### 14. **Backup System Compromise**
- **Target:** HLR backup servers/storage
- **Method:** Attack less-secure backup infrastructure
- **Gains:** Encrypted/plaintext subscriber data
- **Impact:** Offline data breach

#### 15. **Replication Attack**
- **Target:** HLR-VLR replication link
- **Method:** Intercept or inject into database replication stream
- **Gains:** Real-time subscriber data changes
- **Impact:** Data manipulation, eavesdropping

---

### **Protocol Interception and Spoofing**

#### 16. **SS7 Man-in-the-Middle**
- **Target:** Signaling links between databases
- **Method:** Intercept SS7/SIGTRAN traffic
- **Gains:** All MAP messages, subscriber data
- **Impact:** Complete visibility of network operations

#### 17. **Diameter Protocol Attacks (4G/5G)**
- **Target:** HSS (evolved HLR)
- **Method:** Exploit Diameter routing, SCTP vulnerabilities
- **Gains:** Similar to SS7 attacks but on LTE/5G networks
- **Impact:** Modern network compromise

#### 18. **GT (Global Title) Spoofing**
- **Target:** HLR addressing
- **Method:** Fake originating GT in SS7 messages
- **Gains:** Bypass security filters, masquerade as trusted network
- **Impact:** All SS7-based attacks become possible

---

### **Physical and Insider Attacks**

#### 19. **Insider Database Access**
- **Target:** Direct HLR database access
- **Method:** Compromised employee credentials
- **Gains:** Bulk subscriber data export
- **Impact:** Massive privacy breach, data selling

#### 20. **Physical Data Center Attack**
- **Target:** HLR physical servers
- **Method:** Physical access to data center
- **Gains:** Disk cloning, memory dumping
- **Impact:** Complete database compromise

---

## **Real-World Impact Examples**

### **Case 1: German Bank Theft (2017)**
- **Attack:** SS7 HLR lookup + SMS interception
- **Method:** Used `sendRoutingInfo` to locate subscriber, then intercepted 2FA SMS
- **Impact:** €100,000+ stolen from bank accounts

### **Case 2: UAE Spyware Campaign (2019)**
- **Attack:** Location tracking via ATI queries
- **Method:** Mass `AnyTimeInterrogation` queries to HLR
- **Impact:** Thousands of activists/journalists tracked

### **Case 3: Norwegian Network Breach (2020)**
- **Attack:** HLR backup compromise
- **Method:** Attacked backup system, extracted all subscriber data
- **Impact:** 4 million subscriber records leaked

---

## **Attack Vectors Summary Table**

| Attack Category | Target Database | Protocol Used | Difficulty | Prevalence |
|----------------|-----------------|---------------|------------|------------|
| HLR Lookup | HLR | MAP/SS7 | Low | Very High |
| Location Tracking | HLR/VLR | MAP/SS7 | Low | Very High |
| Data Modification | HLR | MAP/SS7 | Medium | Medium |
| Auth Vector Theft | AuC/HLR | MAP/SS7 | Medium | High |
| SQL Injection | HLR (OAM) | HTTP/SQL | Medium | Low (declining) |
| Diameter Attacks | HSS | Diameter | High | Growing |
| Insider Access | All | Direct | Low | Constant |

---

## **Mitigation Measures**

### **Technical Controls**
1. **SS7 Firewalls** - Filter malicious MAP messages
2. **Diameter Edge Agents** - Protect 4G/5G networks
3. **GT Filtering** - Only allow trusted global titles
4. **Rate Limiting** - Detect bulk queries (ATI attacks)
5. **Encryption** - Encrypt all database backups
6. **Multi-factor Authentication** - For OAM access

### **Procedural Controls**
7. **International SS7 Screening** - Filter international signaling
8. **Regular Security Audits** - Check for unauthorized MAP operations
9. **Penetration Testing** - Test SS7/Diameter security
10. **Insider Threat Programs** - Monitor database access

### **Modern Solutions**
11. **Sigtran/SBc Security** - For LTE networks
12. **SEPP (Security Edge Protection Proxy)** - For 5G networks
13. **Blockchain-based Authentication** - Emerging solution
14. **AI/ML Anomaly Detection** - Detect unusual database access patterns

---

## **Example Attack Demonstration**

Let me show you how a simple HLR lookup attack works (for educational purposes):

```python
# Simplified example of SS7 HLR lookup attack
# WARNING: This is for educational purposes only

import socket
import struct

# Craft a fake MAP sendRoutingInfo request
def create_send_routing_info(msisdn):
    # SS7 TCAP/MAP message structure (simplified)
    message = {
        'message_type': 'sendRoutingInfo',
        'target_msisdn': msisdn,
        'fake_gt': '1234567890',  # Spoofed global title
        'request_info': ['subscriberInfo', 'locationInfo']
    }
    return encode_message(message)

# Send to target HLR
def attack_hlr_lookup(msisdn, target_hlr_gt):
    ss7_message = create_send_routing_info(msisdn)
    
    # Send via SS7 network (simplified)
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.connect(('ss7_gateway', 2905))  # SIGTRAN port
    
    sock.send(ss7_message)
    response = sock.recv(4096)
    
    # Parse response
    subscriber_info = decode_response(response)
    print(f"IMSI: {subscriber_info['imsi']}")
    print(f"Current VLR: {subscriber_info['vlr']}")
    print(f"Location: {subscriber_info['cell_id']}")
    
    return subscriber_info

# Attacker gets complete subscriber location
subscriber = attack_hlr_lookup('+1234567890', '724-555-1234')
```

This is why telecom database security is critical - these attacks are real, practical, and happening daily. Modern 5G networks are addressing many of these with improved security architectures (SEPP, encrypted signaling), but legacy SS7 networks remain vulnerable.
