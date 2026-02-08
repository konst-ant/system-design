# Yelp-Like Location Search System – Concise System Design

## 2. Requirements and Goals

### Objective

Design a Yelp-like service that stores information about places and allows users to search for nearby places based on location, with support for reviews and ratings.

### Functional Requirements

* CRUD operations for Places (add, update, delete).
* Geo-based search: find places within a given radius of a user’s latitude/longitude.
* Users can add reviews with:

  * Text
  * Photos
  * Rating

### Non-Functional Requirements

* Low-latency, near real-time search.
* High read throughput: searches vastly outnumber writes.
* Scalability to hundreds of millions of places.
* Fault tolerance and high availability.

---

## 3. Scale Estimation

* Total places: **500 million**
* Search load: **100K QPS**
* Annual growth: **20%** for both places and QPS
* Design must scale horizontally and support future growth without re-architecture.

---

## 4. Database Schema

### Places Table

| Field       | Size      |
| ----------- | --------- |
| LocationID  | 8 bytes   |
| Name        | 256 bytes |
| Latitude    | 8 bytes   | 
| Longitude   | 8 bytes   |
| Description | 512 bytes |
| Category    | 1 byte    |

Taking double for lat/long to be able to store numbers up to 6 precisionn digits in range -90 : 90, -180 : 180 (float only allows 7 digits including precision; double -16 digits)

* Total per record: **793 bytes**
* 8-byte LocationID chosen for long-term growth.

### Reviews Table

| Field      | Size      |
| ---------- | --------- |
| LocationID | 8 bytes   |
| ReviewID   | 4 bytes   |
| ReviewText | 512 bytes |
| Rating     | 1 byte    |

* Reviews and photos stored separately to avoid bloating Place records.
* Photos stored in object storage; DB keeps metadata only.

---

## 5. System APIs

### Search API

```
search(
  api_dev_key,
  search_terms,
  user_location,
  radius_filter,
  maximum_results_to_return,
  category_filter,
  sort,
  page_token
)
```

### Key API Characteristics

* API key used for authentication and throttling.
* Supports filtering by radius, category, and sorting.
* Pagination handled via page tokens.
* Returns JSON with:

  * Name
  * Address
  * Category
  * Rating
  * Thumbnail

---

## 6. Basic System Design and Algorithms

### Core Problem

Efficiently search **hundreds of millions of static geo-points** with low latency.

### Assumption

* Place locations change rarely → index optimized for reads, not frequent updates.

---

### a. SQL-Based Approach (Baseline)

* Store lat/long as indexed columns.
* Query using bounding box:

```
Latitude BETWEEN X-D AND X+D
AND Longitude BETWEEN Y-D AND Y+D
```

**Limitations**

* Independent latitude/longitude indexes return huge result sets.
* Intersection is expensive at large scale.
* Poor performance with 500M rows.

---

### b. Fixed Grid Indexing

#### Idea

* Divide the world into fixed-size grids.
* Each grid contains all places within its boundaries.

#### Benefits

* Search only relevant grids instead of the entire dataset.
* Typically search current grid + 8 neighbors.

#### Storage

* Store `GridID` with each Place.
* Maintain in-memory hash:

  * Key: GridID
  * Value: list of LocationIDs

#### Memory Estimate

* ~20M grids (10-mile radius) = (2 * PI * 6000 miles)^2 
* Index size ≈ **4 GB** = 8 * 20M + 8 * 500M 

#### Problem

* Uneven distribution:

  * Dense cities → overloaded grids
  * Sparse areas → wasted capacity

---

### c. Dynamic Grid Size (QuadTree)

#### Concept

* Split grids dynamically once they exceed **500 places**.
* Dense areas → small grids
* Sparse areas → large grids

#### Data Structure: QuadTree

* Each node represents a geographic region.
* Each node has up to 4 children, splitting parent region into 4 quadrants and quadrants going in known order (e.g.clockwise from 1-st quadrant)
* Each node may have up tp 500 locations before it need to split (sparced nodes will be mergin together)
* Leaf nodes store actual places.

#### Operations

* **Build**: we will start with single node, split up node into 4 quadrants (since it has more than 500), distribute locations among them and recursively continue this process. Note: Consider alternatives (very important) For static spatial data, QuadTree is often not optimal. Better options: Geohash + sorting, Hilbert / Morton (Z-order) curves, R-tree (bulk-loaded), KD-tree
* **Insert**: split node when capacity exceeded.
* **Search**:

  * Traverse tree to locate user’s grid.
  * Expand to neighbors until radius or result count satisfied.
* **Neighbor discovery**:

  * Via parent pointers
  * Or doubly linked leaf nodes

#### Memory Usage

* Location cache (LocationID, long, lat): 24 bytes/place → **12 GB** = 24 * 500M
* ~1M leaf nodes = 500M / 500
* Internal nodes ≈ **10 MB**
* Total ≈ **12.01 GB**, fits in RAM.

#### Insert Workflow

* Insert new Place → DB + QuadTree.
* If partitioned, locate correct server first.

---

## 7. Data Partitioning

### Motivation

* Index may exceed single-machine memory.
* One server cannot handle future QPS.

### a. Region-Based Sharding

**Pros**

* Simple conceptual model.

**Cons**

* Hot regions overload servers.
* Uneven growth across regions.
* Requires frequent repartitioning.

### b. LocationID-Based Sharding (Chosen)

* Hash(LocationID) → server
* Each server maintains its own QuadTree.
* Search:

  * Query all partitions
  * Aggregate nearby results centrally
* Even data distribution.
* Different QuadTree shapes per partition acceptable.

---

## 8. Replication and Fault Tolerance

### Replication Model

* Primary–Secondary per shard.
* Writes go to primary.
* Reads served from secondaries.

### Failure Handling

* Primary failure → secondary promoted.
* Dual failure → rebuild server.

### Fast Rebuild Strategy

* Maintain **QuadTree Index Server**:

  * Maps QuadTree server → set of LocationIDs
  * Stores LocationID + lat/long
* Allows rapid reconstruction without scanning entire DB.
* Index server also replicated.

---

## 9. Cache

* Introduce cache (e.g., Memcached) for hot Places.
* Cache stores full Place details.
* Application servers check cache before DB.
* Eviction policy: **LRU**
* Cache size adjusted based on traffic patterns.

---

## 10. Load Balancing

### Placement

1. Client → Application Servers
2. Application → Backend Servers

### Strategies

* Start with Round Robin.
* Upgrade to load-aware LB for:

  * CPU
  * Latency
  * Queue depth
* LB removes unhealthy servers automatically.

---

## 11. Ranking

### Beyond Distance

* Rank by:

  * Popularity
  * Rating
  * Relevance

### Approach

* Store popularity score per Place.
* Each partition returns top-K results.
* Aggregator merges and sorts globally.

### Updating Popularity

* Popularity changes infrequently.
* Batch updates once or twice daily during low traffic.
* Avoid real-time QuadTree mutations.

---

## Summary

This design achieves:

* Low-latency geo search at massive scale
* Efficient memory usage via QuadTrees
* Horizontal scalability via sharding
* High availability through replication
* Extensible ranking and caching layers

It is optimized for **read-heavy, location-static workloads**, making it well-suited for a Yelp-like system.
