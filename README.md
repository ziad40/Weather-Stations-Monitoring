﻿# Weather-Stations-Monitoring 

DDIA course project about a distributed system that fetches weather information from multiple sources, archives and visualizes it.

![image](https://github.com/basel-bytes/Weather-Stations-Monitoring/assets/95547833/4e67c7df-e938-4d8a-8351-ee81214cd39d)

## Agenda
- [System Architecture](#system-architecture)
- [How to Run](#how-to-run)

<a name="-system-architecture"></a> 
## System Architecture
There are 3 major components implemented using 6 microservices. </br>
The 3 major components are:
1) [Data Acquisition](#data-acquisition).
2) [Data Processing & Archiving](#data-processing-and-archiving).
3) [Data Indexing](#data-indexing).

<a name="-data-acquisition"></a> 
## Data Acquisition
Multiple weather stations that feed a queueing service (Kafka) with their readings.</br>
### A) Weather Stations
- Implemented in [weather-station service](./weather-station/)
- We have 2 types of stations: **Mock Stations**, and **Adapter Stations**.
- Mock Stations are required to randomise its weather readings.
- Adapter Stations get results from Open-Meteo according to a latitude and longitude given at the beginning.
- Both types will have the same **battery distribution(30% low - 40% medium - 30% high)** and dropping percentage of **10%**.
- We use a **station factory design pattern** to indicate which type of station we would like to build.
- We use a **builder design pattern** to build the message to be sent.
- Messages coming from Open Meteo are brought according to the **Adapter and filter integration pattern**.
![image](https://github.com/basel-bytes/Weather-Stations-Monitoring/assets/95547833/e8e84e9f-32f2-4a34-9f7d-85640fe94b1d)




### B) Kafka Processors
- Implemented in [kafka-processing service](./kafka-processor/).
- There are two types of processing following **router integration pattern**:
  
| Processing Type | Description |
| --- | --- |
| Dropping Messages | Processes messages by some probabilistic sampling, then throw some of them away |
| Raining Areas | Processes messages and detects rain according humidity, then throw messages from rainy areas to some topic. We use Kstream and filters to do such processors |

- All these processing produce undropped messages to **weather_topic** And all records have **humidity >= 70** go to **raining topic**.


<a name="-data-processing-and-archiving"></a> 
## Data Processing And Archiving
- This is done in 2 stages:

| Point | Description | Implementation |
| --- | --- | --- |
| 1) Data Collection and Archiving | Where data is consumed from Kafka topics then written in batches of 10k messages in the form of parquet files to support efficient analytical queries. | [base-central-station service](./base-central-station/) |
| 2) Parquet files Partioning | Where a **Chronological job** scheduled to run every 24 hours at midnight **with the help of k8s** and **implemented in Scala using Apache Spark** to partition messages in parquet files by **both day and station_id**. | [sparky-file-partition service](./sparky-file-partition/) |
<p align="center">
  <img src="https://github-production-user-asset-6210df.s3.amazonaws.com/95547833/252986467-6c78a981-bee9-4202-9866-64ed193e124d.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20230713%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20230713T135057Z&X-Amz-Expires=300&X-Amz-Signature=c3c34f118066e8c220f61757bbd328060cfd13666eb3cc0b1b9d13e0b63fe34e&X-Amz-SignedHeaders=host&actor_id=95547833&key_id=0&repo_id=641528715" alt="Partitioned Parquet Files">
</p>
    
<a name="-data-indexing"></a> 
## Data Indexing
This is composed of two parts: </br>
a) [BitCask Storage](#bitcask-storage) </br>
b) [Elastic Search and Kibana](#elastic-search-and-kibana) </br>

<a name="-bitcask-storage"></a> 
## Bitcask Storage
- Implemented in [bitCask](./base-central-station/src/main/java/bitcask/).
- We implemented the BitCask Riak LSM to maintain an updated store of each station status as discussed in [the book](https://drive.google.com/file/d/120RsgrUsgNFkg1hChAG05LI6LS09sM-p/view?usp=drive_link) with some notes:
  *  **Scheduling compaction** over segment files to avoid disrupting active readers.
  *  **No checksums implemented** to detect errors.
  *  **No tombstones implemented** for deletions as there is no point in deleting some weather station ID entry. We just deleted Replica files on Compaction.
  *  here is the structure of entry in segment files: </br>

      | **ENTRY:** | timestamp | key_size | value_size | key| value |
      | --- | --- | --- | --- | --- | --- |
      | **SIZE:** | 8 bytes | 4 bytes | 4 bytes | key_size | value_size |
    
  *   ### Classes
        | Class | Description |
        | --- | --- |
        | BitCask | Provides API for writing, reading keys, and values. |
        | ActiveSegment | Handles the currently active segment file. |
        | Compaction Task | Contains the following two runnable classes and uses a timer schedule to run them periodically. |
        | Compact |  Used to compact segments periodically. |
        | deleteIfStale | Used to delete a file if it's marked as a stale file (a file that was compacted). |
        | BitCaskLock | Used to ensure one writer at a time for the active file. |
        | SegmentsReader | Given the hash table entry and file segment path, returns the value corresponding to this key. |
        | SegmentsWriter | Given key, value, segment file path, writes the key, value into the segment. |
        | Pointer | Points to an entry and contains two attributes: filePath, ByteOffset. |

  * ### Crash Recovery Mechanism
    - Create a new hashMap.
    - Read Active file, start to end, and add key, pointer to value pairs to hashTable.
    - Read hint files, from end to start, and fill hashTable with key value pairs.
  * ### Compaction Mechanism
    - Loop on all replica files, read each replica file from start to end, add its key value pairs to hashMap.
    - Loop on hashTable, write each key value as entry in a compacted file. 
    - Mark Replica File as **Stale**.
    - **Another scheduled task deleteIfStale will delete file** replica file **if it’s STALE file**. 
  * ### MultiWriter concurrency Mechanism
    Implemented using **WriteLockFile**, to allow for concurrency over multiple bitCask Instances, on the same shared Storage. 
    - When Writer acquires lock, **WriteLockFile** is created in segments path.
    - When another writer wants to write to active file, it **checks** for **WriteLockFile**, if it **exists**, it **waits until file is deleted**.
    - When the other **writer finishes**, it **deletes WriteLock file**. 
 

<a name="-elastic-search-and-kibana"></a> 
## Elastic Search and Kibana
- Implemented in [elastic-search-and-kibana](./elastic-search-and-kibana/).
- We implemented a **Python script** listening to kafka topic (paths_topic) where the base central station **sends paths of newly-created parquet files**.
- The **script then a loads a parquet file** and converts it to records and **connects to elasticsearch to upload records**.
- Here is the Kibana visualisations confirming Battery status distribution of some stations confirming the battery distribution of stations:
![image](https://github.com/basel-bytes/Weather-Stations-Monitoring/assets/95547833/5adae40f-c1cf-41d9-b4b5-e62c031b20e2)

- Here is also Kibana visualisations calculating the percentage of dropped messages from stations confirming the required percentage 10%:
![image](https://github.com/basel-bytes/Weather-Stations-Monitoring/assets/95547833/86d3962b-6ca2-42a2-a430-531b725e5787)


<a name="-how-to-run"></a> 
## How To Run
This project is designed to be deployed on Kubernetes, so you should follow the following steps to run it: 
1) Build a docker image from the dockerfile in each service with the following names:

    | Service | Image Name: Version|
    | --- | --- | 
    | [weather-station](./weather-station/) | weather-station-image:latest|
    | [kafka-processor](./kafka-processor/) | kafka-processor-image:latest |
    | [base-central-station](./base-central-station/) | base-central-station-image:latest |
    | [sparky-file-partition](./sparky-file-partition/) | sparky-file-partition-image:latest |
    | [elastic-search-and-kibana](./elastic-search-and-kibana/) | elastic-loader-image:latest |
  
2) Use the following command to load the docker images into k8s: </br>
  `minikube image load <image name>`
3) Apply all k8s YAML files in [k8s directory](./k8s/).
   
