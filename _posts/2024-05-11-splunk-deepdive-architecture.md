---
layout: post
title: "Advanced Splunk Cluster Architecture & Disaster Recovery: The Definitive Textbook Guide"
date: 2024-05-11 09:00:00 +0900
categories: splunk architecture security
---

# Chapter 1: The Evolution of Splunk Architecture

In the early days of big data, Splunk relied on a tightly coupled architecture where compute and storage were inseparable. This model, known as Direct Attached Storage (DAS), required organizations to scale their indexer count linearly with their data retention needs. If you wanted to keep a year of data, you needed enough indexers to house that data on local disks, even if your search load didn't justify the extra CPU and RAM.

Today, Big Tech environments handle tens of petabytes daily. The DAS model is no longer sustainable. This chapter explores the modern Splunk architecture, focusing on SmartStore and Multisite Indexer Clustering, which decouple compute from storage and provide robust disaster recovery (DR) capabilities.

---

# Chapter 2: SmartStore Architecture Deep Dive

SmartStore is the most significant architectural shift in Splunk's history. It moves the "Master" copy of data from the indexer's local disk to a remote object store (like Amazon S3, Google Cloud Storage, or Azure Blob Storage).

## 2.1 The Decoupled Model
In a SmartStore-enabled cluster, the indexer's role changes from a "storage node" to a "compute and cache node."

1.  **Compute Layer (Indexers):** Responsible for data ingestion, parsing, indexing, and searching. They use high-performance NVMe SSDs as a transient cache.
2.  **Storage Layer (Remote Object Store):** Acts as the persistent, authoritative repository for all warm and cold buckets. It provides 99.999999999% durability and scales independently of the indexers.

## 2.2 The Cache Manager: The Brain of the Indexer
The Cache Manager is a service running on each indexer that manages the local disk space. It ensures that the most relevant data is available for searches while staying within the bounds of the local SSD capacity.

### The Caching Lifecycle
*   **Hot Buckets:** When data is first ingested, it's written to "Hot" buckets on the local NVMe SSD. These buckets are not yet in the remote store.
*   **Rolling to Warm:** When a hot bucket reaches its size or age limit, it "rolls" to warm. At this moment, the indexer initiates an upload to the remote object store.
*   **The "Master" Transition:** Once the remote store acknowledges the upload, the remote copy becomes the authoritative "Master." The local copy is now just a cached version.
*   **Eviction (LRU):** When the local disk reaches a threshold (defined by `max_cache_size`), the Cache Manager identifies buckets to evict. It uses a Least Recently Used (LRU) algorithm, prioritizing the removal of buckets that haven't been searched recently.
*   **On-Demand Fetching:** If a search requires a bucket that has been evicted, the Cache Manager fetches it from the remote store. It's intelligent: it might only download the `.tsidx` (index) files first to see if the bucket contains relevant results before downloading the larger raw data files.

## 2.3 Performance Tuning for SmartStore
To maintain high performance, administrators must monitor the **Cache Hit Ratio**. A low hit ratio indicates that the local cache is too small, forcing frequent downloads from S3, which introduces latency.

*   **`max_cache_size`:** The total space allocated for the cache.
*   **`hotlist_recency_secs`:** A setting that prevents recently rolled buckets from being evicted for a specific duration, ensuring that "fresh" data is always local.

---

# Chapter 3: Multisite Indexer Clustering & Disaster Recovery

For mission-critical environments, a single-site cluster is a single point of failure. If a data center loses power or connectivity, your data becomes inaccessible. Multisite Indexer Clustering solves this by distributing data across multiple geographic locations.

## 3.1 Site Awareness and Search Affinity
Each component in the cluster (Indexers, Search Heads, Cluster Manager) is assigned a `site` ID (e.g., `site1`, `site2`).

*   **Search Affinity:** Search Heads are configured to prefer indexers in their own site. This minimizes cross-site network traffic, reducing latency and egress costs.
*   **Replication Factor (RF) and Search Factor (SF):** In a multisite environment, these settings are site-aware. For example, a `site_replication_factor = origin:2, total:3` ensures that two copies exist at the site where the data originated, and three copies exist across the entire cluster.

## 3.2 SmartStore and RF/SF
In SmartStore, the rules for RF and SF change. Since the remote store handles the durability of warm buckets, RF and SF primarily apply to **Hot** buckets.
*   **Rule:** In SmartStore, `RF` must equal `SF`. There is no benefit to having a non-searchable copy of a hot bucket in a decoupled environment.

---

# Chapter 4: DR Failover Scenario Walkthrough

Let's walk through a real-world disaster scenario in a two-site (Site A and Site B) configuration.

### Phase 1: Normal Operations
*   Data is ingested into both sites.
*   Site A Search Heads query Site A Indexers.
*   Site B Search Heads query Site B Indexers.
*   Warm buckets are uploaded to a shared or replicated S3 bucket.

### Phase 2: The Disaster (Site A Failure)
1.  **Detection:** The Cluster Manager (CM) detects that all indexers in Site A are unreachable.
2.  **Promotion:** The CM immediately promotes the "searchable" copies of hot buckets in Site B to "primary" status.
3.  **Search Redirection:** Search Heads in Site B (or surviving SHs in Site A) automatically redirect all queries to the healthy indexers in Site B.
4.  **SmartStore Fetching:** If a search requires data that was previously only cached in Site A, the Site B indexers will fetch that data from the remote object store. The search continues with minimal interruption.

### Phase 3: Recovery (Site A Returns)
1.  **Re-joining:** When Site A indexers come back online, they contact the CM.
2.  **Metadata Sync:** The indexers synchronize their metadata. They don't need to perform a massive "bucket sync" of raw data because the warm/cold data is already safe in the remote store.
3.  **Cache Rehydration:** As users start searching from Site A again, the indexers will gradually "rehydrate" their local NVMe caches by fetching frequently accessed buckets from S3.

---

# Chapter 5: Search Head Clustering (SHC)

While indexers handle the data, Search Head Clusters handle the user experience.

*   **The Captain:** One search head is elected as the Captain. It coordinates job scheduling and replicates search artifacts (the results of previous searches) across the cluster.
*   **Member Nodes:** All other search heads are members. If the Captain fails, a new one is elected automatically.
*   **Artifact Replication:** This ensures that if a user's session moves from one search head to another, their search history and results follow them.

---

# Chapter 6: Best Practices for Big Tech Environments

1.  **Network Latency:** Ensure that the round-trip time (RTT) between sites is within Splunk's recommended limits (typically < 100ms for multisite). High latency can cause the Cluster Manager to incorrectly mark sites as down.
2.  **S3 Security:** Use IAM roles and bucket policies to restrict access. Enable encryption at rest (SSE-S3 or SSE-KMS).
3.  **Monitoring:** Use the Splunk Monitoring Console (MC) to track SmartStore upload/download rates and cache eviction frequency.
4.  **Compression:** Enable compression for site-to-site data replication to save bandwidth.

---

# Chapter 7: Self-Assessment & Review

**Question 1:** Why must `RF` equal `SF` in a SmartStore environment?
*   **Answer:** Because the remote store provides the ultimate durability for warm data. Redundant non-searchable copies of hot data provide no additional value in a decoupled architecture.

**Question 2:** What happens to a search if the required data is not in the local NVMe cache?
*   **Answer:** The Cache Manager fetches the necessary files from the remote object store. It prioritizes metadata and index files to minimize the amount of data downloaded.

**Question 3:** How does Search Affinity improve performance?
*   **Answer:** It keeps search traffic local to the data center, avoiding the latency and cost of cross-site data transfers.

---

This concludes our deep dive into Splunk's advanced architecture. By mastering SmartStore and Multisite Clustering, you can build systems that are not only cost-effective but also resilient enough to withstand entire data center outages.
