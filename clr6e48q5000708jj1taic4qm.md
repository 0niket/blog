---
title: "Moving large objects from Database to File System - part 1"
datePublished: Tue Jan 09 2024 13:31:18 GMT+0000 (Coordinated Universal Time)
cuid: clr6e48q5000708jj1taic4qm
slug: moving-large-objects-from-database-to-file-system
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/fteR0e2BzKo/upload/20db87a14b7bb943e5c24dd1dbdb931a.jpeg
tags: distributed-system, amazon-efs

---

## **Background**

Help Center's one of the main features is to display Frequently Asked Questions (FAQs). These FAQs can be edited using a Rich Text Editor, external APIs, and file uploads. FAQs are nothing but HTML documents that are ultimately rendered on the web and in-app web views.

In our primary database, Mongo, the FAQ HTML documents are stored alongside other fields such as FAQ title, search keywords, published status, schedules, and more. With support for 185 different languages, there can be a maximum of 185 HTML documents.

However, this setup started causing issues. Queries became slower, and memory consumption on Mongo nodes increased, affecting the rest of the application.

## **Causes**

During the postmortem analysis, we identified three main issues:

1. Lack of size limit on HTML documents.
    
2. No restrictions on elements that can be added, such as Base64 images.
    
3. Denormalised schema design.
    

## **Concerns**

Addressing issues such as adding a limit on FAQ size and reaching a consensus on the types of supported HTML elements required discussions with the customer and various stakeholders within the organization. Overall, this process was expected to be time-consuming.

## **Options**

To address the burning issue and reduce the blast radius during oncall situation, we decided to move the HTML documents out of the collection.

Although Mongo supports GridFS as a file system solution, it didn't fit our use case.

According to the [Mongo documentation](https://www.mongodb.com/docs/manual/core/gridfs/):

> Do not use GridFS if you need to update the content of the entire file atomically. As an alternative, you can store multiple versions of each file and specify the current version in the metadata. Update the metadata field indicating the "latest" status in an atomic update after uploading a new version of the file, and remove previous versions if needed.

Previously, we used AWS S3 as the default option for object storage, but it wasn't suitable due to response time concerns.

After careful consideration, we opted for AWS Elastic File System as it aligns well with our requirements:

1. Sub-millisecond response time for both reads and writes.
    
2. High availability as it is a distributed file system with replicas and backup facilities.
    
3. Cost-efficient
    

## Solution

There needed a way to co-relate files on EFS with the entry in DB. At a time, there could be only one HTML document corresponding to individual translations of the FAQ. For this, we decided to consider properties of FAQ like domain, FAQ ID, language code, and updated-at.

The directory structure we finalized looked somewhat like this:

`/faqs/<domain>/<faq-id>/<lang-code>-<updated-at>.html`

Adding updated-at to the file path was mainly required to handle atomic writes

This is how the directory structure would look after migrating existing FAQ documents to the file system.

```yaml
- faqs
    - domain-1
        - faq-id-1
            - en-<updated_at_t1>.html
            - en-<updated_at_t2>.html
        - faq-id-2
            - en-<updated_at>.html
            - jp-<updated_at>.html
    - domain-2
        - faq-id-1
            - en-<updated_at>.html
            - es-<updated_at>.html
```

### Handling Writes

Filesystems do not inherently ensure atomic writes. For instance, if a write operation to a file, such as `en.html`, is interrupted due to a process crash, the file may end up in an inconsistent state. Moreover, any read requests during the ongoing write process might retrieve inaccurate data.

To address this issue and guarantee consistent reads and writes, we have implemented a solution based on the [compare-and-update](https://en.wikipedia.org/wiki/Compare-and-swap) approach. This atomic operation is instrumental in synchronizing data between MongoDB and the Elastic File System (EFS).

Our strategy involves using a timestamp, labeled as `updated_at`, at the application layer. This timestamp ensures that we are updating the same record that was accessed during the initial data read from the database and EFS. Through this method, we effectively prevent data corruption and consistently hold either entirely old or entirely new information. Also, the compare-and-update approach makes sure that files and documents in db are not updated concurrently.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1704806683449/21a20094-1bc1-421f-b6cb-1c4060024042.png align="center")

Following are the details of the algorithm:

1. FAQ service’s HTTP API gets called for updating an FAQ.
    
2. FAQ service cancels the write operation if document with `updated_at` value is not found. An appropriate error code/message is returned in the response to the API call.
    
3. If the FAQ service finds a document corresponding to `updated_at` value, it means that it has not been updated in the meantime and there are no chances of corrupting data.
    
4. Following the above operation, it evicts the cache, updates FAQ document, updates HTML files on EFS, and finally responds with a success message in the API call.
    

### Handling Reads

The aforementioned approach, incorporating the use of the `updated-at` timestamp during both file and document writes, effectively mitigates concurrency issues during read operations. Let's examine the potential concurrency problem scenarios:

1. User X initiates a request to read an FAQ.
    
2. Simultaneously, User Y requests to write to the same FAQ.
    
3. User X reads from MongoDB.
    
4. User Y writes to the document, updates the Elastic File System (EFS), and completes the transaction.
    
5. User X attempts to read from EFS.
    

Consequences:

Without the use of the `updated-at` timestamp, let's consider how this scenario could impact the read operation. User X reads the document from the database at step 3. Meanwhile, User Y writes to the same document and updates files on EFS at step 4. Suppose the modified file is `en.html`. Now, at step 5, User X reads `en.html` from EFS. Consequently, User X possesses a document containing mixed data—a blend of older metadata fetched from MongoDB and newer file versions.

Now, let's revisit the scenario where the `updated-at` timestamp is employed to version the files on EFS. With this approach, each write operation generates a new file on EFS. User X reads the document from the database at step 3, while User Y writes to the same document and updates files on EFS at step 4. Assuming the modified file is `en-t2.html`, at step 5, User X reads the file for the `en` language using the `updated_at` timestamp received in step 2. Therefore, the file fetched is `en-t1.html`. In essence, the data acquired by User X is not corrupted, as it combines metadata from the database and files from EFS of the same version. This ensures data consistency and prevents the retrieval of mixed or outdated information.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1704806780591/599ead79-ebe1-4e0f-a018-26ffa8c6abe3.png align="center")

Following are the details of the read algorithm:

1. FAQ service gets a read HTTP API request
    
2. FAQ service first checks if the document corresponding to the FAQ ID is cached in the key-value store.
    
3. If the document is found in the cache then it is returned
    
4. Otherwise, the FAQ document is fetched from Mongo, files are fetched from EFS. Both results are merged, cached, and returned in the HTTP API call.
    

Note: It’s an existing architectural decision to evict cache on write paths and cache documents on read paths.

### Managing Dangling Files Cleanup

When associating files with the `updated_at` timestamp, it's inevitable to encounter dangling files that require periodic deletion. This cleanup process can be treated as a scheduled monthly task.

Cleanup process:

1. Iterate through each FAQ in the `FAQs` collection.
    
2. Access the corresponding FAQ translations stored in the Elastic File System (EFS).
    
3. Examine the `updated_at` timestamp for the language translation in the FAQ document.
    
4. Delete all other `<language-code>-<updated_at>.html` files, except for the one that aligns with the `updated_at` timestamp in the language translation data of the MongoDB document.
    

Implementation Provisions

To ensure the smooth execution of this cleanup process, the following provisions should be considered:

* **Scheduled Execution:** Set up a cron job to run this cleanup algorithm every month.
    
* **Manual Trigger:** Provide a mechanism to manually trigger the cleanup job, offering flexibility in addressing specific cleanup requirements.
    
* **Domain-Specific Trigger:** Incorporate a functionality to trigger the cleanup job selectively for a particular domain. This feature ensures faster processing and allows tailored cleanup based on specific domain needs.
    

## Operability changes

1. For every FAQ service node, we need EFS to be mounted on the EC2 instance. This means while spawning a new FAQ service instance, it needs to have EFS mounted on it before the application starts serving the requests.
    
2. EFS multi-zone standard setup had good durability guarantees.
    
3. Files should be backed up on S3 similar to any other database.
    

Such data migration is a risky affair. In the [next blog post](https://blog.anikethendre.dev/moving-large-objects-from-database-to-file-system-part-2), we'll examine how the whole project was managed and rolled out.