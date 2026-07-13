# How Uber Finds Nearby Drivers In Real Time

## 1. Uber

- On-demand ride-hailing platform
- Connects riders with nearby drivers in real-time
- Available in 10,000+ cities globally
- Millions of rides completed daily

---

## Functional Requirements

- Riders can request a ride by providing pickup and destination location
- System finds all nearby available drivers and matches them with the driver having least ETA
- Riders can see the assigned driver's live location as they approach
- Drivers periodically send location updates
- Drivers can accept/reject a ride request
- Once ride is complete rider can see the entire route of ride

## Non-Functional Requirements

- Rider matching latency < 300ms
- Strong consistency for ride matching
- Eventual consistency for location updates
- System must be highly available
- System should be able to work during peak traffic
- System must be able to handle 10% user growth each year.

---

## Assumptions & Scale

### User Assumptions:

- MAU: 180M
- DAU: 20% → 36M DAU
- Daily trips (2 per active user): 72M

### Driver Assumptions:

- 1.2M Daily Active Drivers

### Ride Requests:

- 72M trips/day ≈ 834 requests/sec average
- Peak (×10): ~8,500 QPS

### Location Updates → Every 5 Seconds

- 1 sec → 0.2 updates requests
- 1.2M active driver * 0.2 updates requests
- 240,000 updates/second

---

## Storage

| Entity | Stored In |
|--------|-----------|
| User | PostgreSQL |
| Driver | PostgreSQL |
| Ride | PostgreSQL |
| Driver Location | Redis |
| Driver Availability | Redis |
| Trip Location History | Cassandra |

### Storage Notes:

- **Active Drivers: 1 Million** → Live Data: In-memory processing crucial for real-time
- **Driver Location Updates: Write-Intensive** → 240,000 Writes/sec → Requires high-throughput write store (for history) & fast updates (for real-time index)
- **Ride Requests: Read-Intensive** → 8,500 Reads/sec → Low-latency read store for spatial queries
- **Rides per Day: 72 Million** → Scalable transactional store for trip records

---

## Storage Requirements

### Row Size Calculations

- **Rider:** 4 + 50 + 50 + 100 + 20 + 200 + 8 + 8 = 440 bytes
- **Driver:** 4 + 50 + 50 + 100 + 20 + 50 + 20 + 8 + 8 + 8 = 318 bytes
- **Location:** 4 + 4 + 4 + 8 + 8 + 8 = 32 bytes
- **Ride:** 4 + 4 + 4 + 8 + 8 + 8 + 8 + 8 + 8 + 8 + 20 + 8 + 8 = 112 bytes

### Table Size

- **Ride:** 112 bytes × 26,280,000,000 rides = 2,741.46 GB
- **Rider:** 440 bytes × 180,000,000 users = 73.75 GB
- **Driver:** 318 bytes × 5,000,000 drivers = 1.48 GB
- **Location:** 32 bytes × 1,000,000 active drivers = 0.03 GB

---

## Growth Estimates

Assuming 10% Growth Each Year

| Year | Rides (GB) | Riders (GB) | Drivers (GB) | Locations (GB) | Cumulative Storage (GB) |
|------|------------|-------------|--------------|----------------|-------------------------|
| 1 | 2,741.46 | 73.75 | 1.48 | 0.03 | 2,816.72 |
| 2 | 3,015.61 | 81.13 | 1.63 | 0.03 | 5,915.12 |
| 3 | 3,317.17 | 89.24 | 1.79 | 0.04 | 9,323.36 |
| 4 | 3,648.89 | 98.16 | 1.97 | 0.04 | 13,072.42 |
| 5 | 4,013.78 | 107.98 | 2.17 | 0.04 | 17,196.39 |

---

## High-Level Design (HLD)

```mermaid
flowchart TB
    subgraph Clients
        R[ Rider App]
        D[ Driver App]
    end
    
    subgraph Edge
        AG[ API Gateway]
        LB[ Load Balancer]
    end
    
    subgraph Services
        RS[ Ride Service]
        MS[ Matching Service]
        LS[ Location Service]
        NS[ Notification Service]
    end
    
    subgraph Cache
        RC[( Redis Cluster)]
    end
    
    subgraph Databases
        PG[( PostgreSQL)]
        CS[( Cassandra)]
    end
    
    R --> AG
    D --> AG
    AG --> LB
    LB --> RS
    LB --> MS
    LB --> LS
    LB --> NS
    
    RS --> PG
    MS --> RC
    LS --> RC
    LS --> CS
    NS --> RC
```

---

## API Endpoints

### 1. Request Ride
```
POST /rides
Request: {
  "pickup": {"lat": 24.8607, "lng": 67.0011},
  "destination": {"lat": 24.9056, "lng": 67.0822}
}
```

### Flow1: Ride Request

```mermaid
sequenceDiagram
    participant Rider
    participant API as API Gateway
    participant RS as Ride Service
    participant MS as Matching Service
    participant Redis
    participant Postgres
    participant Driver

    Rider->>API: POST /rides (pickup, destination)
    API->>RS: Forward request
    RS->>Postgres: Create ride record (PENDING)
    RS->>MS: Request driver matching
    MS->>Redis: Query nearby available drivers (GEO search)
    Redis-->>MS: Return list of nearby drivers
    MS->>MS: Calculate ETA for each driver
    MS-->>RS: Return best driver match
    RS->>Postgres: Update ride with driver assignment
    RS-->>API: Return ride details with driver info
    API-->>Rider: Ride assigned
    RS->>Driver: Send ride request notification
    
    Driver->>API: POST /rides/{ride_id}/decision
    API->>RS: Forward decision
    RS->>Postgres: Update ride status
    RS->>Redis: Update driver availability
```

---

### 2. Driver Accept/Reject
```
POST /rides/{ride_id}/decision
{
  "driver_id": "uuid",
  "decision": "ACCEPT" | "REJECT"
}
```

### 3. Start Ride
```
POST /rides/{ride_id}/start
```

### 4. Complete Ride
```
POST /rides/{ride_id}/complete
Response: {
  "ride_id": "uuid",
  "status": "COMPLETED",
  "route_summary": {
    "distance_km": 12.4,
    "duration_sec": 1520
  }
}
```

---

## Sequence Diagram

```mermaid
sequenceDiagram
    participant Rider
    participant API as API Gateway
    participant RS as Ride Service
    participant MS as Matching Service
    participant LS as Location Service
    participant Redis
    participant Cassandra
    participant Postgres
    participant Driver

    Note over Driver,Redis: Flow 2: Location Updates (Continuous)
    loop Every 5 seconds
        Driver->>API: POST /location/update (lat, lng)
        API->>LS: Forward location update
        LS->>Redis: Update driver location (sync)
        LS->>Cassandra: Write location history (async)
    end

    Note over Rider,Driver: Flow 1: Ride Request
    Rider->>API: POST /rides (pickup, destination)
    API->>RS: Create ride request
    
    RS->>MS: Find nearby drivers
    MS->>Redis: GEO search for drivers near pickup
    Redis-->>MS: Return nearby drivers
    
    MS->>MS: Calculate best match (min ETA)
    MS-->>RS: Return matched driver
    
    RS->>Postgres: Create ride record
    RS-->>API: Return ride details
    API-->>Rider: Ride assigned with driver info
    
    RS->>Driver: Send ride request
    
    Driver->>API: POST /rides/{ride_id}/decision (ACCEPT)
    API->>RS: Process acceptance
    RS->>Postgres: Update ride status (ACCEPTED)
    RS->>Redis: Mark driver unavailable
    
    Note over Rider,Driver: Ongoing Location Updates During Ride
    loop Every 5 seconds
        Driver->>API: /location/update
        API->>LS: Forward update
        LS->>Redis: Update (sync)
        LS->>Cassandra: Write history (async)
    end
    
    loop Every 2 seconds
        Rider->>API: GET /rides/{ride_id}/location
        API->>LS: Get driver location
        LS->>Redis: Query driver location
        Redis-->>LS: Return location
        LS-->>API: Return location
        API-->>Rider: Display driver location
    end
    
    Driver->>API: POST /rides/{ride_id}/start
    API->>RS: Start ride
    RS->>Postgres: Update status (STARTED)
    
    Driver->>API: POST /rides/{ride_id}/complete
    API->>RS: Complete ride
    RS->>Postgres: Update status (COMPLETED)
    RS->>Redis: Mark driver available
    RS-->>API: Return route summary
    API-->>Rider: Ride complete
```

---

## Flow 2: Location Updates

### Scale
- 72M rides/day
- 240 updates/ride
- ~17.3B rows/day

### Goal
- No large or hot partitions
- Fast route reads
- Horizontal scaling
- Complete ride route (using Cassandra)
- Redis keeps the latest location

```mermaid
flowchart LR
    subgraph Driver
        D[Driver App]
    end
    
    subgraph Edge
        AG[API Gateway]
    end
    
    subgraph Services
        LS[Location Service]
    end
    
    subgraph Cache
        RC[(Redis<br/>Latest Location)]
    end
    
    subgraph Database
        CS[(Cassandra<br/>Location History)]
    end
    
    D -->|Location Update| AG
    AG --> LS
    LS -->|Sync Write| RC
    LS -->|Async Write| CS
```

---

## Ride Request & Matching: Key Decisions

### Why Redis?

- In-memory, sub-millisecond geo queries
- Handles massive concurrent reads/writes
- Horizontally scalable via sharding

### GeoHash vs Quadtree

- Quadtree requires rebalancing on movement
- High write amplification at scale
- GeoHash supports O(1) location updates
- Natively supported by Redis

### Consistency & Availability

- **Strong Consistency:** Atomic Writes
- **TTLs:** Prevent stale driver availability
- **Redis:** Real time coordination
- **Postgres:** ACID ride lifecycle

```mermaid
flowchart TB
    subgraph Matching Flow
        Request[Ride Request] --> GeoSearch[Redis GEO Search]
        GeoSearch --> NearbyDrivers[Nearby Drivers List]
        NearbyDrivers --> CalculateETA[Calculate ETA for Each]
        CalculateETA --> BestMatch[Select Best Match]
        BestMatch --> Reserve[Reserve Driver Atomically]
    end
    
    subgraph Redis Operations
        GEOADD[GEOADD: Add/Update Location]
        GEOSEARCH[GEOSEARCH: Find Nearby]
        SETNX[SETNX: Atomic Reserve]
        TTL[TTL: Auto-Expire Availability]
    end
```

---

## Driver Location Updates: Key Decisions

### Cassandra: High Volume Writes

- Massive write throughput + append only
- Time series data
- Masterless + no SPOF

### Rider Updates

- **Websockets:** Persistent connections
- **Streaming VS Polling:** Low latency + high frequency updates

### Cassandra Partitioning Strategy

**Schema:**
- **PK:** ((ride_id, event_date), timestamp)
- **Partition:** ride_id + event_date
- **Clustering:** timestamp ASC

### Why ride_id?

- One ride = one bounded time-series
- ~240 rows/partition (<<100k limit)
- 72M partitions/day → evenly distributed
- TTL (e.g., 90 days) caps storage

```mermaid
flowchart TB
    subgraph Cassandra Table Structure
        PK[Partition Key: ride_id, event_date]
        CK[Clustering Key: timestamp ASC]
        
        subgraph Partition[One Partition = ride_id + event_date]
            Row1[Timestamp: 10:00:00 - Location 1]
            Row2[Timestamp: 10:00:05 - Location 2]
            Row3[Timestamp: 10:00:10 - Location 3]
            RowN[Timestamp: 10:05:00 - Location N]
        end
        
        PK --> Partition
        CK --> Partition
    end
    
    subgraph Benefits
        B1[~240 rows/partition - well under limit]
        B2[Evenly distributed across nodes]
        B3[TTL auto-cleans old data]
        B4[Fast range queries by time]
    end
```

---

## Scaling & Load Management

### Horizontal Scaling

- **Stateless Services:** API Gateway, Location Service, Matching Service
- **Redis + Cassandra:** Distribute load & inherent fault tolerance

### Vertical Scaling

- **PostgreSQL:** Early stage + simplicity for ACID

### Data Durability & Recovery

**Redis Cluster:**
- 1 Master + 1 Replica per shard

**Cassandra:**
- Replication Factor = 3
- WRITES → CL = QUORUM
- READS → CL = ONE
- Peer-to-Peer Replication

```mermaid
flowchart TB
    subgraph Frontend
        AG1[API Gateway 1]
        AG2[API Gateway 2]
        AG3[API Gateway N]
    end
    
    subgraph Services
        LS1[Location Service 1]
        LS2[Location Service 2]
        MS1[Matching Service 1]
        MS2[Matching Service 2]
    end
    
    subgraph Redis Cluster
        R1[(Redis<br/>Shard 1)]
        R2[(Redis<br/>Shard 2)]
        R3[(Redis<br/>Shard 3)]
        R1R[(Redis<br/>Replica)]
        R2R[(Redis<br/>Replica)]
        R3R[(Redis<br/>Replica)]
    end
    
    subgraph Cassandra Cluster
        C1[(Node 1)]
        C2[(Node 2)]
        C3[(Node 3)]
    end
    
    subgraph PostgreSQL
        PG[(PostgreSQL<br/>Master)]
        PGR[(PostgreSQL<br/>Replica)]
    end
    
    AG1 --> LS1
    AG1 --> MS1
    AG2 --> LS2
    AG2 --> MS2
    
    LS1 --> R1
    LS1 --> R2
    LS2 --> R2
    LS2 --> R3
    
    MS1 --> R1
    MS2 --> R3
    
    LS1 --> C1
    LS1 --> C2
    LS2 --> C2
    LS2 --> C3
    
    RS1[Ride Service 1] --> PG
    RS2[Ride Service 2] --> PG
    PG --> PGR
```

---

## Appendix

### GeoHashing

```mermaid
flowchart LR
    subgraph World[World Map]
        Grid[Grid Division]
    end
    
    subgraph GeoHash[GeoHash Encoding]
        Lat[Latitude: 24.8607]
        Lng[Longitude: 67.0011]
        Binary[Binary Conversion]
        Interleave[Interleave Bits]
        Base32[Base32 Encoding]
        
        Lat --> Binary
        Lng --> Binary
        Binary --> Interleave
        Interleave --> Base32
        Base32 --> Result[GeoHash: t4245...]
    end
    
    subgraph Properties
        P1[Longer shared prefix = closer together]
        P2[Reverse not guaranteed!]
    end
```

**Key Point:** Longer the shared prefix between two geohashes, the spatially closer they are together. The reverse of this is not guaranteed!

### Quad Trees

```mermaid
flowchart TB
    subgraph Quadtree[Quadtree Structure]
        Root[Root - Whole Region]
        Q1[Quadrant 1 - NW]
        Q2[Quadrant 2 - NE]
        Q3[Quadrant 3 - SW]
        Q4[Quadrant 4 - SE]
        
        Root --> Q1
        Root --> Q2
        Root --> Q3
        Root --> Q4
        
        Q1 --> Q1A[Sub-Quadrant]
        Q1 --> Q1B[Sub-Quadrant]
        Q1 --> Q1C[Sub-Quadrant]
        Q1 --> Q1D[Sub-Quadrant]
    end
    
    subgraph Issues
        I1[Requires rebalancing on movement]
        I2[High write amplification at scale]
    end
```

---

## Appendix: Replication and Quorums in Cassandra

```mermaid
flowchart TB
    subgraph Write[Write Operation]
        WC[Client] -->|Write| Coordinator
        Coordinator -->|Write to RF=3| N1[Node 1]
        Coordinator -->|Write to RF=3| N2[Node 2]
        Coordinator -->|Write to RF=3| N3[Node 3]
        N1 -.->|Acknowledged| Coordinator
        N2 -.->|Acknowledged| Coordinator
        N3 -.->|Acknowledged| Coordinator
        Coordinator -->|Success after QUORUM| WC
    end
    
    subgraph Read[Read Operation]
        RC[Client] -->|Read| ReadCoord[Coordinator]
        ReadCoord -->|Read from| RN1[Node 1]
        ReadCoord -->|Read from| RN2[Node 2]
        RN1 -.->|Digest| ReadCoord
        RN2 -.->|Digest| ReadCoord
        ReadCoord -->|Compare & Return| RC
    end
    
    subgraph Quorum[Quorum Settings]
        RF[Replication Factor = 3]
        W[WRITES: CL = QUORUM = 2]
        R[READS: CL = ONE = 1]
        
        W --> WP[Write ensures durability with 2 acks]
        R --> RP[Read returns fastest response]
    end
```