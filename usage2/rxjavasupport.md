# RXJavaSupport

RXJava support in DBFlow is an _incubating_ feature and likely to change over time. We support RXJava3 only and have made the extensions + DBFlow compatibility almost identical - save for the changes and where it makes sense in each version.

Currently it supports 

1. Any `ModelQueriable` can be wrapped in a `Single`, `Maybe`, or `Flowable` \(to continuously observe changes\). 

2. Single + `List` model `save()`, `insert()`, `update()`, and `delete()`. 

3. Streaming a set of results from a query 

4. Observing on table changes for specific `ModelQueriable` and providing ability to query from that set repeatedly as needed.

## Getting Started

Add the separate packages to your project:

```groovy
dependencies {

  // RXJava3
  compile "com.github.agrosner.dbflow:reactive-streams:${dbflow_version}"

}
```

## Wrapper Language

We can convert wrapper queries into different kinds of RX operations.

For a single query:

Using vanilla transactions:

```kotlin
(select 
  from MyTable::class)
  .async(db) { d -> queryList(d) }
  .execute { _, r ->}
```

Using RX:

```kotlin
(select 
  from MyTable::class)
  .async(db) { d -> queryList(d) }
  .asSingle()
  .subscribeBy { list ->  
    // use list
  }
```

### Observable Queries

We can easily observe a query for any table changes \(once per transaction\) while it is active and recompute via:

```kotlin
(select 
  from MyTable::class)
  .asFlowable { db -> queryList(db) }
  .subscribeBy { list ->  
    // use list
  }
```

This works across joins as well.

## Model operations

Operations are as easy as:

```kotlin
Person(5, "Andrew Grosner")
  .rxInsert(db) // rxSave, rxUpdate, etc.
  .subscribeBy { rowId -> 
  
  }
```

## Query Stream

We can use RX to stream the result set, one at a time from the `ModelQueriable` using the method `queryStreamResults()`:

```kotlin
(select from TestModel::class)
   .queryStreamResults(db)
   .subscribeBy { model -> 

   }
```

