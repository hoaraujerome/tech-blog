+++
date = '2021-12-14T15:27:32-04:00'
title = 'DynamoDB: 3 ways to use the API'
+++

The way you interact with DynamoDB is usually with AWS SDK, where you can perform:

1.  Items-based actions: Anytime you act on a single item - writing, updating, or deleting - you are using an item-based action. You must provide the entire primary key.
    
2.  Query: Read-only actions that allow you to fetch multiple items in a single request. You must provide the partition key and optionally provide sort key conditions.
    
3.  Scan: Full table scan that looks at every item in your table. Avoid it unless you are doing an export or ETL. It's an expensive operation at scale regarding how long it takes to respond to a request and how much capacity you need to service it. Remember that there is a 1MB limit when reading items from the table.
    

As you can see, the SDK is more API Driven than query-driven as it would be in a relational database.
