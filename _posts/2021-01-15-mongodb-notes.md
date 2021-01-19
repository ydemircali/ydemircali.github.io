---
title: MongoDB Beginner Notes
tags: [HowTo, Notes, MongoDB, NoSQL]
style: fill
color: danger
description: Inntroduction to MongoDB.
---

![](https://github.com/ydemircali/ydemircali.github.io/blob/master/_posts/mongodb_logo.png?raw=true)

Source: [MongoDB Tutorial for Beginners](https://www.youtube.com/watch?v=GtD93tVZDX4&list=PLS1QulWo1RIZtR6bncmSaH8fB81oRl6MP)

## Intro

First distinction:

- Relational databases (i.e. MySQL, PostgreSQL, MS SQL Server): they have tables, rows, columns and a PK-FK relational structure relating data one another. Joins are necessary to put data together.
- Non-Relational databases (i.e. NoSQL databases): they mainly store documents, and lack relations.

MongoDB is a NoSQL database. It has no relations, but binary json files not related to each other (BSON).
Here are its main characteristics:

- simple query language (vs standardized SQL)
- data is not related to each other through relations (vs PKs, FKs and joins)
- it stores groups of binary json documents (BSON), called collections (vs tables)
- analogies:
    - collection=table
    - document=row
    - field=column
- schemaless collections (vs strict information added and typed columns)

## Download and Install

    brew install MongoDB

or

    manually download from website
    unzip and place the extracted folder (maybe renamed) in <mongodir>
    add <mongodir>/bin to $PATH
    create <datadir> (default: /data/db, can be customized for instance to <mongodir>/data/db)
        if custom <datadir>, alias mongod --dbpath <customdatadir>

## GUIs for MongoDB:

- dbeaver (open source, free)
- mongochef / studio3T (pay for it)
- robomongo / robo3T (free)

## Database structure

database -> collections -> documents -> fields

## Basic Instructions

### Create Database

    use <dbname>

> If dbname is already present, switch to it. Otherwise create it, then switch to it.

### See current working db

    db

> Note: db is the alias for the database we are using!

### List all databases

    show dbs 

> Only databases with at least some data are displayed.

### Insert documents

    db.<mycollection>.insert([key-value-pairs])
    # where
    [key-value-pairs] = {"name":"Max"}  # json-like!

### Drop / Delete db

    use <db_to_drop>
    db.dropDatabase()
    
### Create collection

    db.createCollection("mycollection", [maybe options])

> Creates an empty collection named "mycollection" in the current db.

    db.mycollection2.insert({"name":"max"})

> If collection "mycollection2" is not present, creates it. Then, insert the document.

### See collections in the current db

    show collections
    
> It lists "mycollection" (empty) and "mycollection2" (having 1 document).

### Drop collections

    db.mycollection.drop()
    
> Collection "mycollection" gets dropped.

### Insert documents into databases

School database example:

- 1 db -> school
- 1 collection -> group of all the school students
- many documents -> 1 for each student in the school

To insert a single document:

    use school
    db.createCollection("students")
    db.students.insert(
        {
            "name":"max",
            "surname":"reds",
            "age":"10"
        }
    )

> Everything in the curly brackets is ONE DOCUMENT (aka a json element).
> 
> Each document has key-value pairs related to the very same student.
> 
> We have one document, with several fields, for each student.
> 
> The group of students is the "collection" (collection of documents).

To insert multiple documents at once:

    db.students.insert([
        {
            "name":"lou",
            "surname":"greens",
            "age":"11"
        },
        {
            "name":"william",
            "surname":"yellows",
            "age":"10"
        },
        {...}
    ])

> Same as before, but add square brackets; separate documents with comma. 

**Note:** when a document is inserted, MongoDB creates a unique identifier ("_id"), unless you put it in the document.

## Query data

    db.<collection>.find()[.pretty()]  # all the values of all the documents within the collection
    db.<collection>.findOne()  # first document in the collection
    
    db.<collection>.find(
        {"studentNo" : "2"}  # where condition (equal)
    )
    db.<collection>.find(
        {"age" : {$gt : "15"}}  # where condition (gt, gte, lt, lte, ne)
    )
    
    db.<collection>.find(
        {"firstName" : "Mark", "age" : "10"}}  # where condition (and of conditions -- with comma)
    )
    db.<collection>.find(
        {$or : [{"firstName" : "Mark"}, {"age" : "15"}]}  # where condition (or of conditions)
    )
    db.<collection>.find(
        {"firstName" : "Mark", $or : [{"age" : "16"}, {"age" : "15"}]}}  # where condition (mixed conditions)
    )

## Update documents (or multiple documents)

General instruction:

    db.<collection>.update(
        {<conditions>},  # how to filter documents
        {<values>}  # which values to update
    )

Examples:

    db.<collection>.update(
        {"_id" : ObjectId("123456")},
        {$set : {"LastName" : "Masen"}}
    )

    db.<collection>.update(
        {"age" : "16"},    # only the 1st document matching the condition!!
        {$set : {"LastName" : "Wogh"}}
    )

    db.<collection>.update(
        {"age" : "16"},    
        {$set : {"LastName" : "Wogh"}},
        {multi : true}    # all the (multiple) documents matching the condition!!
    )

A particular way of inserting / updating documents:

    db.<collection>.save(
        {
            "name":"carl",
            "surname":"browns",
            "age":"10"
        }
    )

> If not present, adds it (insert). If present, set the current values (update). 
> 
> Match is done by "_id".

## Delete / Remove document

    db.<collection>.remove()  # remove all the documents in the collection
    db.<collection>.remove(   # remove documents matching the conditions
        {
            # conditions here
        }
    )
    
Examples:
    
    db.<collection>.remove(   # remove single document (match is done with unique identifier)
        {
            "_id" : ObjectId("2345555")
        }
    )

    db.<collection>.remove(   # remove multiple documents (all matching conditions)
        {
            "age" : "16"
        }
    )
    
    db.<collection>.remove(   # remove N document (the first N matching conditions)
        {
            "age" : "16"
        }, 1  # <-- in this case, N=1 and only the first document with age=16 is deleted
    )

## Projections

To select only necessary fields rather than all the fields of a document.

    db.<collection>.find({},{KEY:1})  # 1: true, 0: false -> do you want to see the KEY field?

Examples:

    db.<collection>.find({},{"LastName":1})  # only _id and LastName (if not specified, _id is always shown)
    db.<collection>.find({},{"LastName":1, "_id":0})  # only LastName

## Limit, sort, skip

Limit: to only select the top N records

    db.<collection>.find({},{"StudentNo":1, "FirstName":1, "_id": 0}).limit(4)
    
> The top-4 students, displayed as (Number, FirstName)

Skip: to skip the first N records
    
    db.<collection>.find({},{"StudentNo":1, "FirstName":1, "_id": 0}).skip(2)  
    
> All the students but the first 2, displayed as (Number, FirstName)
 
Sort: to order records according to the given fields

    db.<collection>.find({},{"StudentNo":1, "FirstName":1, "_id": 0}).sort({"FirstName": 1})  
    
> 1 = ascending order; -1 = descending order

Mixed example:
  
    db.<collection>.find({},{"StudentNo":1, "FirstName":1, "_id": 0})
        .sort({"FirstName": -1})
        .skip(2)
        .limit(3)
    
> Order all the students by descending first name, and take the 3rd, 4th and 5th ones.

## Indexing

Indexed access to efficiently access sparse and large databases.

To add an index on field "myfield"

    db.<collection>.ensureIndex({"myfield" : 1})

To remove an index on field "myfield"

    db.<collection>.dropIndex({"myfield" : 1})

Example:

    # insert 1000000 documents
    for(i=0; i < 1000000; ++i) {
        db.<collection>.insert({"student_id": i, "Name": "Mark"});
    }
	
    # retrieve single(multiple) element(s)
    db.<collection>.findOne({"student_id": 1000})  #immediate! (just one, the first matching)
    db.<collection>.find({"student_id": 1000})  #slow! (many documents might match, check them all)

    # add an index on student_id
    db.<collection>.ensureIndex({"student_id" : 1})
    
    # rerun the query
    db.<collection>.findOne({"student_id": 1000})  #immediate! (as before)
    db.<collection>.find({"student_id": 1000})  #immediate! (now it is indexed)
    
**Note:** do **not** add an index for every field, but just for unique ones!

## Aggregation

Groups values from different documents and perform a variety of operations on the grouped data. 

**Return a single value** after the aggregation (e.g. select count(*) from ... group by ...  -> in this case the aggregated value is the count).

    db.<collection>.aggregate(  # group by gender, aggregation op = count.
        [
            {$group : {_id : "$Gender", GenderCount : {$sum : 1}}}  # others are sum, min, max, avg, count, ...
        ]
    )

    db.<collection>.aggregate(  # group by gender, aggregation op = max.
        [
            {$group : {_id : "$Gender", MaxAge : {$max : "$Age"}}}  # others are sum, min, max, avg, count, ...
        ]
    )

    db.<collection>.aggregate(  # group by gender, aggregation op = min.
        [
            {$group : {_id : "$Gender", MinAge : {$min : "$Age"}}}  # others are sum, min, max, avg, count, ...
        ]
    )

## Backup and Restore

    mongodump  # backup command
    mongorestore  # restore command
    
    # Files are backed up and restored from <mongodir>/bin/<dump_folder> folder
    <mongodir>
    | - ...
    | - bin
    |    |- ...
    |    |- dump
    |    |   |- database_1
    |    |   |   |- collection_1.bson
    |    |   |   |- collection_2.bson
    |    |   |- database_2
    |    |   |   |- collection_1.bson

### For whole databases

    mongodump  # backup all the databases     
    mongodump --db <database_name>  # for selective backup

    mongorestore  # restore all the databases
    mongorestore --db <database_name> /path/to/<database_name>  # for selective restore

### For specific collections of a database

    mongodump --db <database_name> --collection <collection_name>
    mongorestore --db <database_name> --collection <collection_name> /path/to/<collection_name>.bson
