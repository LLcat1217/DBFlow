---
description: >-
  DBFlow provides a flexible way to observe changes on models and tables in this
  library.
---

# Observability

By default, DBFlow supports direct [model notification](observability.md#direct-changes) via `DirectModelNotifier`.

Also, the `DBFlowDatabase` class provides a `TableObserver` object, which provides observability backbone for `QueryDataSource` \(Paging extensions\), `LiveData`, and `RXJava3` support.

Also with configuration, DBFlow can use the [`ContentResolver`](https://developer.android.com/reference/android/content/ContentResolver.html) to send changes through the android system. We then can utilize [`ContentObserver`](http://developer.android.com/reference/android/database/ContentObserver.html) to listen for these changes via the `FlowContentObserver`

## Direct Changes

We must use the shared instance of the `DirectModelNotifier` since if we do not, your listeners will not receive callbacks.

Next register for changes on the `DirectModelNotifier`:

```kotlin
DirectModelNotifier.get().registerForModelChanges(User::class, object: ModelChangedListener<User> {
            override fun onModelChanged(model: User, action: ChangeAction) {
                // react to model changes
            }

            override fun onTableChanged(action: ChangeAction) {
              // react to table changes.
            }
        })
```

Then unregister your model change listener when you don't need it anymore \(to prevent memory leaks\):

```kotlin
DirectModelNotifier.get().unregisterForModelChanges(User::class, modelChangedListener);
```

### TableObserver

The `TableObserver` is a more efficient mechanism for tracking table-level changes on the DB. It utilizes triggers under the hood to track table changes during operations \(but only with active Table observers used\). It is useful for providing reactive queries that will autoreload when data changes.

**Note:** You must execute a db transaction to automatically check for table changes in observers.

```kotlin
// wire up the observer (example uses LiveData)
(select from MyTable::class
  where name.like("Andrew%"))
  .liveData { db, q -> q.queryList(db)
  .observe(lifecycle) { value ->
  
  }
  
// perform some transaction
(update<MyTable>() set name.eq("Andrew") where name.eq("Mike"))
  .async(db) { d -> execute(d) }
  .execute { _, result -> 
    Log.w("MyTable", "Updated ${result} rows")
  }
```

To manually access it \(when you need to update it outside of a transaction\):

```kotlin
database<AppDatabase>().tableObserver.checkForTableUpdates()
```

### FlowContentObserver

To use a `FlowContentObserver` instead of direct changes, we override the default `ModelNotifier`:

```kotlin
FlowManager.init(FlowConfig.Builder(context)
            .database(DatabaseConfig.Builder(TestDatabase.class)
                .modelNotifier(ContentResolverNotifier(context, authority))
                .build()).build())
```

The content observer converts each model passed to it into `Uri` format that describes the `Action`, primary keys, and table of the class that changed.

A model:

```kotlin
@Table(database = AppDatabase.class)
class User(@PrimaryKey var id: Int = 0, @Column var name: String = "")
```

with data:

```kotlin
User(55, "Andrew Grosner").delete()
```

converts to:

```text
dbflow://%60User%60?%2560id%2560=55#DELETE
```

Then after we register a `FlowContentObserver`:

```kotlin
val observer = FlowContentObserver()

observer.registerForContentChanges(context, User::class)

// do something here
// unregister when done
observer.unregisterForContentChanges(context)
```

### Model Changes

It will now receive the `Uri` for that table. Once we have that, we can register for model changes on that content:

```kotlin
observer.addModelChangeListener(object : OnModelStateChangedListener {
  override fun onModelStateChanged(table: Class<*>?, 
        action: ChangeAction,
        primaryKeyValues: Array<SQLOperator>) {
    // do something here
  }
})
```

The method will return the `Action` which is one of: 

1. `SAVE` \(will call `INSERT` or `UPDATE` as well if that operation was used\) 

2. `INSERT` 

3. `UPDATE` 

4. `DELETE`

The `SQLOperator[]` passed back specify the primary column and value pairs that were changed for the model.

If we want to get less granular and just get notifications when generally a table changes, read on.

### Register for Table Changes

Table change events are similar to `OnModelStateChangedListener`, except that they only specify the table and action taken. These get called for any action on a table, including granular model changes. We recommend batching those events together, which we describe in the next section.

```kotlin
addOnTableChangedListener(object : OnTableChangedListener {
    override fun onTableChanged(tableChanged: Class<*>?, action: ChangeAction) {
        // perform an action. May get called many times! Use batch transactions to combine them.
    }
})
```

### Batch Up Many Events

Sometimes we're modifying tens or hundreds of items at the same time and we do not wish to get notified for _every_ one but only once for each _kind_ of change that occurs.

To batch up the notifications so that they fire all at once, we use batch transactions:

```kotlin
val observer = FlowContentObserver()
observer.registerForContentChanges(context, User::class.java)

db.beginTransactionAsync(users.fastInsert().build())
    .execute(ready = { observer.beginTransaction() },
    completion = { observer.endTransactionAndNotify() })
```

Batch interactions will store up all unique `Uri` for each action \(these include `@Primary` key of the `Model` changed\). When `endTransactionAndNotify()` is called, all those `Uri` are called in the `onChange()` method from the `FlowContentObserver` as expected.

If we are using `OnTableChangedListener` callbacks, then by default we will receive one callback per `Action` per table. If we wish to only receive a single callback, set `setNotifyAllUris(false)`, which will make the `Uri` all only specify `CHANGE`.

## 

