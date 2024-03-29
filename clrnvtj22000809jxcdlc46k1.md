---
title: "Deciding indexes"
datePublished: Sun Jan 21 2024 19:18:56 GMT+0000 (Coordinated Universal Time)
cuid: clrnvtj22000809jxcdlc46k1
slug: deciding-indexes
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/T92YFbo07-Q/upload/bc7a11f927b32b1eb807313c9c857bd1.jpeg
tags: databases, distributed-system, indexing

---

## Database Basics

There are two main functionalities of a database. One is to write to the database and the other is to read from the database.

For simplicity, consider a database as a file where each line corresponds to a record.

During the reading process, the system is required to traverse each record and assess whether it aligns with the specified filters. As the database expands and accumulates a substantial volume of records, this approach becomes inefficient and has a linear time complexity.

Indexes address this challenge by introducing another data structure known as a BTree, with the B+ Tree variant being widely favored for its efficiency. The BTree is organized into pages, each typically sized around 4kb. A root page serves as the initial reference point, leading to subsequent child pages, ultimately reaching a page containing values instead of references. BTrees are often designed as shallow trees, allowing for rapid record retrieval and it has logarithmic time complexity.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1705863917446/9be55b4e-72d4-4b94-a462-1378dabb8836.png align="center")

## Let’s look at an Example

Consider we have a SaaS application where we onboard different customers and allow them to publish web pages. Users can select and add forms to a specific web page from a predefined set of forms. To accomplish this task, the application incorporates a linking feature. This feature facilitates the association of a chosen form with a particular web page, providing users with a seamless and intuitive method for enhancing their pages with the desired forms.

### Page

A web page has set properties like title, body, and call to action.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1705896767653/fe4683a1-d777-4c4a-a451-870ec0fa2e3d.png align="center")

### CTA → Form

Clicking on the call to action i.e. “Contact Us” button opens the form

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1705896706120/72d15298-81b4-4151-a08d-78723249d640.png align="center")

### Linking form → page

Using the linking feature forms can be linked to the page

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1705896857868/5420c211-0b85-4679-be77-34614364f0bc.png align="center")

## Data model

For this example, I’m using Mongo DB. However, a similar process can be followed for other relational and non-relational databases.

We’ll build a generic data model using which any two entities can be linked. For now, let’s call it `entity_linking` collection. In this collection, we’ll have the following fields.

| Field | Data Type | Description |
| --- | --- | --- |
| id | string | This uniquely identifies the association. |
| customer\_id | string | Since ours is a SaaS application. We need customer\_id to filter the data related to individual customers. |
| entity\_id | string | id of the entity which will be referred to at other places. In this case, it is form because it’ll be referenced at other places like page. |
| type | string | This indicates the type of association or relation between two entities. Example: **form\_to\_page** |
| referenced\_at | Array of page ids\[“page-id-1“, “page-id-2“, …\] | A place where an entity is being referenced. |
| data | Object | the data field can have any properties. In the case of, page to form linking, it’ll have cta\_text and fallback\_text properties. |
| data.cta\_text | string | The CTA text field can be a free-flowing text. This is a call to action to open the form. |
| data.fallback\_text | string | Fallback text is displayed if the form can not be rendered for any reason. |

## What are Query patterns for the collection?

1. Update CTA and/or fallback text for the form **(write query)**
    
2. Link form with pages **(Write query)**
    
3. Get CTA and fallback text for the form **(Read query)**
    
4. Get CTA and fallback text for the page **(Read query)**
    

As we can see queries **#3** and **#4** are the read queries so we’ll see how well our Mongo queries perform to fetch results for these queries. We’ll use `explain` mongo query to do this.

## How to read Explain result?

There are four things that we need to look at in the result returned by the `explain` function.

1. **Stage** - Full collection scan(`COLLSCAN`) or index scan(`IXSCAN`). There are other stages as well like `SORT` and Mongo will use this stage if query result needs to be sorted.
    
2. **nReturned** - Number of documents that match the query condition and are returned in the result.
    
3. **totalKeysExamined** - Number of index entries scanned
    
4. **totalDocsExamined** - Number of documents examined during query execution. Refers to the total number of documents examined and *not* to the number of documents returned. For example, a stage can examine a document to apply a filter. If the document is filtered out, then it has been examined but will not be returned as part of the query result set. If a document is examined multiple times during query execution, `totalDocsExamined` counts each examination. That is, `totalDocsExamined` is *not* a count of the total number of *unique* documents examined. This is an expensive operation as data might not be present in-memory and disk seek might be required.
    

## Benchmarking indexes

We’ll take simple queries to get CTA and fallback text for the form. For this we’ll run the following Mongo find query —

```jsx
db.entity_linking.explain("executionStats").find({
  customer_id: "customer_id_1",
  type: "form_to_page" 
  entity_id: "form_id_1"
});
```

This is what we get in the execution stats

```jsx
"nReturned" : 1,
"totalKeysExamined" : 0,
"totalDocsExamined" : 12,
"stage" : "COLLSCAN",
```

**stage** is `COLLSCAN` which means that it’s a collection scan which is not a good idea.

**totalKeysExamined** is 0 which means there is no index data structure which was scanned for this query.

**totalDocsExamined** is 12 which means it actually read 12 documents and checked if filter is matching with the document.

As we can see results are pretty bad! We are going through each document and trying to see if it matches the query. For running this query, it just went through a single-stage scan but the query planner can choose to do a multi-stage scan of the database to return results in which case **totalDocsExamined** can go up and it is bad because this means more IO operations.

Let’s add an index for the first parameter. so our index will be `customer_id_1`. customer\_id is the key and 1 represents ascending order. if we say `customer_id_-1` then it indexes data in descending order.

Let’s run the same query again and this time we get the following:

```jsx
"nReturned" : 1,
"totalKeysExamined" : 4,
"totalDocsExamined" : 4,
"stage" : "IXSCAN",
```

This is still not the best because it still examined 4 keys and had to go through all 4 documents.

To improve this further, we’ll drop the earlier index and create a new one - `customer_id_1_type_1`

Now, it’ll first match the domain and then it’ll also match the type, and then only it’ll try to access the document for scanning.

```jsx
"nReturned" : 1,
"totalKeysExamined" : 2,
"totalDocsExamined" : 2,
"stage" : "IXSCAN"
```

It still scanned 2 documents while only 1 document was returned so it can be optimised further.

Now, we’ll create an index with the following `customer_id_1_type_1_entity_id_1` and run the explain query.

```jsx
"nReturned" : 1,
"totalKeysExamined" : 1,
"totalDocsExamined" : 1,
"stage" : "IXSCAN"
```

This is optimal because `nReturned == totalKeysExamined == totalDocsExamined`

In this case, we were trying to find exactly one document and hence every parameter should have a value of 1.

Note: The index we have created is a compound index rather than a simple index. The benefit of a compound index is that the index will be first sorted by customer\_id then type and then by entity\_id. This performs well for the query patterns we have listed above.

## Pros of using indexes

Faster read / search operations

## Cons of using indexes

For every write operation, `B+ tree` the data structure needs to be updated by often taking a lock to avoid concurrency issues. This makes write operations more complex.

## Conclusion

* Look for query patterns for your collection. I.e. look for how users are going to use your application.
    
* Create a data model as per your application requirements.
    
* See if read query patterns are optimally returned by the database. If not, experiment with different keys to find out.
    
* Be **mindful** while creating indexes as write operations become more costly.
    
* Though this is specific to Mongo, similar methods can be applied to any database.