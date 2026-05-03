+++
date = '2026-05-03T15:14:05-04:00'
draft = true
title = 'Dynamodb Data Modeling Draft'
+++

3 basic steps:

1.  Start with an ERD (Entity Relationship Diagram) : what entities do I have in my application and how do they relate to each other (one to one, one to many, etc...)
    
2.  Define your access patterns : how are you gonna use this application and how we are gonna fetch these entities, what are our access patterns
    
3.  Design ur primary key & secondary indexes
    

GSI / Secondary Indexes https://www.youtube.com/watch?v=DIQVJqiSUkE

https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/SecondaryIndexes.html
