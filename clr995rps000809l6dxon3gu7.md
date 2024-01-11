---
title: "Managing risks with data migration projects"
datePublished: Thu Jan 11 2024 13:35:50 GMT+0000 (Coordinated Universal Time)
cuid: clr995rps000809l6dxon3gu7
slug: managing-risks-with-data-migration-projects
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/NXt5PrOb_7U/upload/a36c7110fb6ff7029632a8b3ed0ccfe9.jpeg

---

This post serves as a continuation of the previous article on [Moving large objects from database to file system](https://blog.anikethendre.dev/moving-large-objects-from-database-to-file-system). Here, we will explore strategies for navigating the risks associated with data migration projects while ensuring timely deliveries.

Data migrations entail significant risks, such as data loss, data corruption, and performance impacts. The severity of each risk varies, requiring tailored solutions. For instance, data loss or corruption can have severe consequences, whereas efficiency and performance-related issues can be managed and eventually rectified through diverse methods.

## Divide & Conquer

The objective is to access FAQ HTML documents from EFS and the remaining metadata from the database, creating a composite document by merging results from both sources. This goal is achieved through dual reads and writes initially, gradually transitioning away from this approach as confidence in the system builds.

This was achieved in four phases

* Phase 1: Synchronous read & write to database, Async write to EFS
    
* Phase 2: Synchronous read & write to database, Async read & write to EFS
    
* Phase 3: Synchronous read & write to EFS, Async write to database
    
* Phase 4: Synchronous read & write to EFS
    

In the first phase, database is the primary source so we read from it and write to it in primary thread. write to EFS can be done in a separate thread. The reason behind doing it asynchronously is that we want to ensure that the write operation works and don’t want to unnecessarily increase the latency of all write operations.

In the second phase, we have dual reads & writes to database & EFS. Here as well, we are keeping reads and writes to EFS as asynchronous as we simply want to roll out the code, measure impact, and have it working before actually adding it in the code path.

As we gain confidence in the new system, we completely move to having EFS as primary source. We only write to the database now for the sake of backup.

As our confidence in new system becomes stronger, we only have EFS as source for reads and writes.

## Measure twice & cut once

Building confidence in the system requires robust data backing it up. To achieve this, logging, monitoring, and alerting mechanisms are crucial. The key observability aspects include the following:

* EFS is not available — This is a severe error. It was logged and an alert was raised. Also, the HTTP service that served FAQs did not start if EFS was unavailable which means it made a lot of necessary noise.
    
* Not able to locate file for reading — Logs error in Kibana and increments counter for the corresponding graph on Grafana.
    
* Reading file failed after retries — Logs error in Kibana and increments counter for the corresponding graph on Grafana. It also raised an alert based on the following formula, `(no. of failures within x mins > threshold value)`
    
* Write failed due to the same updated\_at timestamp — as we saw in the previous blog post, there can be concurrent updates for the same FAQs. write operation would fail in that case and we would add an error log to Kibana and increment the corresponding counter on Grafana.
    
* Writing file failed after retries — If writes failed after retries then event will be logged and alert will be raised based on the following formula, `(no. of failures within x mins > threshold value)`
    

Latency measurements for all EFS read and write functions are conducted, and performance impact is assessed through plotted graphs after each phase release. This iterative approach ensures a thorough understanding of performance implications before advancing to the next phase.

## Conclusion

The phased approach to project division and comprehensive observation of system-level changes prove instrumental in effectively managing potential risks.