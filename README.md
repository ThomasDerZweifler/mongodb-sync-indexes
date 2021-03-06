# mongodb-sync-indexes

Synchronize the indexes of **mongodb collections** with a list of index objects, structured as the [mongo shell output](http://docs.mongodb.org/manual/reference/method/db.collection.getIndexes). Only the differences are applied and logged.

# Installation

```
npm i mongodb-sync-indexes
```

# Usage

```javascript
var syncIndexes = require('mongodb-sync-indexes');

var eventHandler = syncIndexes(indexList, collection, [options], [callback]);

// or

var eventHandler = syncIndexes(indexListMap, db, [options], [callback]);
```

**Arguments:**

- **indexList**: Array of indexes.

- **indexListMap**: An object with collection names as keys and indexLists as values.

- **collection**: The mongodb collection to synchronize indexes. 

- **db**: The db containing the collections to synchronize indexes.

- **options**: Optional object. Configuration of the synchronization. Possible options :
      - *log*: Boolean. Default: true. Logs all events with useful informations.
      - *verbose*: Boolean. Default: false. Logs all index information on creation/deletion.

- **callback**: Optional function. Called at the end of synchronization.

**Returns:**
- **eventEmitter**: events: 
      - *createIndex*: Before creating an index.
      - *dropIndex*: Before dropping an index.
      - *createdIndex*: When the index is successfully created.
      - *droppedIndex*: When the index is successfully dropped.
      - *done*: Fired at the end of execution. If a callback is provided in the arguments, it is attached to this event.

# Examples

- Updating a collection

```javascript
var assert = require("assert"),
    syncIndexes = require('mongodb-sync-indexes');
    
// Connection URL
var url = "mongodb://localhost:27017/test";
    
// You can also store this structure in a .json file
var indexList = [
    {
        "key": {
            "importantField": 1
        },
        "unique": true
    },
    {
        "key": {
            "anotherField": 1
        },
        "name": "Heisenberg",
        "sparse": true,
        "w": 1
    }
];

MongoClient.connect(url, function(err, db) {
      assert.equal(err, null);
      
      collection = db.collection("IAmGoingBackTo505");
      
      // We create an index that's not in the indexList: it'll be dropped.
      collection.createIndex({country_name: 1}, function(err) {
            assert.equal(err, null);
            
            syncIndexes(indexList, collection);
      }
});
```

In the console you'll see the following:
```
Dropping index {"country_name":1} in collection IAmGoingBackTo505...
Done. Index dropped has name country_name_1

Creating index {"importantField":1} in collection IAmGoingBackTo505...
Done. Index created has name importantField_1

Creating index {"anotherField":1} in collection IAmGoingBackTo505...
Done. Index created has name Heisenberg

Finished synchronization.
```

Finally, in your collection, the indexes are stored like this:

```
[
    {
        "key": {
            "_id": 1
        },
        "name": "_id_",
        "ns": "test.IAmGoingBackTo505",
        "v": 1
    },
    {
        "key": {
            "importantField": 1
        },
        "name": "importantField_1",
        "ns": "test.IAmGoingBackTo505",
        "unique": true,
        "v": 1
    },
    {
        "key": {
            "anotherField": 1
        },
        "name": "Heisenberg",
        "ns": "test.IAmGoingBackTo505",
        "sparse": true,
        "v": 1
    }
]
```

See how: 

1) Mongodb automatically creates the main index, the one with key {"_id": 1};

2) When not specified, the name is also automatically created using the information available;

3) When specified, the name provided is used;

4) Mongodb automatically adds the properties "ns" and "v", which are ignored in our comparisons;

5) The property "w", defined in mongodb as the write concern (a guarantee that MongoDB provides when reporting on the success of a write operation), is not a property to be stored in the index itself;

6) Be careful when defining the properties predefined in mongodb: their types must be respected (name must be a string, etc);

7) No need to bother with mongodb subtitilities, such as: necessity of creating the collection to access its indexes, impossibility of dropping the main index, etc. This is all taken care of.

8) We don't pass any errors to the callback. Use the "error" event.

- Updating a database

```javascript
var assert = require("assert"),
    syncIndexes = require('mongodb-sync-indexes');
 
// Connection URL
var url = "mongodb://localhost:27017/test"; 
 
// You can also store this structure in a .json file
var indexListMap = {
    "BreakingBad": [
        {
            "key": {
                "I AM THE ONE WHO KNOCKS": 1
            },
            "name": "Heisenberg",
            "unique": true
        },
        {
            "key": {
                "SAY MY NAME": 1
            },
            "name": "whoami"
        }
    ],
    "Tinder": [
        {
            "key": {
                "geospatialIndex": 1
            },
            "sparse": true,
            "dropDups": false,
            "w": 1,
            "min": 10,
            "max": 20,
            "expireAfterSeconds": 1
        }
    ]
};

MongoClient.connect(url, function(err, db) {
      assert.equal(err, null);
            
      syncIndexes(indexListMap, db);
});
```

Notice how we map the name of each collection we want to synchronize with the array of indexes desired.

In the console you'll see the following:
```
Creating index {"I AM THE ONE WHO KNOCKS":1} in collection BreakingBad...
Done. Index created has name Heisenberg

Creating index {"SAY MY NAME":1} in collection BreakingBad...
Done. Index created has name whoami

Creating index {"geospatialIndex":1} in collection Tinder...
Done. Index created has name geospatialIndex_1

Finished synchronization.
```

Finally, in the collection "BreakingBad" you'll see this:

```
[
    {
        "key": {
            "_id": 1
        },
        "name": "_id_",
        "ns": "test.BreakingBad",
        "v": 1
    },
    {
        "key": {
            "I AM THE ONE WHO KNOCKS": 1
        },
        "name": "Heisenberg",
        "ns": "test.BreakingBad",
        "unique": true,
        "v": 1
    },
    {
        "key": {
            "SAY MY NAME": 1
        },
        "name": "whoami",
        "ns": "test.BreakingBad",
        "v": 1
    }
]
```

Meanwhile, in the collection "Tinder":

```
[
    {
        "key": {
            "_id": 1
        },
        "name": "_id_",
        "ns": "test.Tinder",
        "v": 1
    },
    {
        "expireAfterSeconds": 1,
        "key": {
            "geospatialIndex": 1
        },
        "min": 10,
        "max": 20,
        "name":  "geospatialIndex_1",
        "ns": "test.Tinder",
        "sparse": true,
        "v": 1
    }
]
```

# Using the EventEmitter returned

The code 

```javascript
syncIndexes(indexList, collection, {log: false}, callback);
```

is equivalent to:

```javascript
var eventHandler = syncIndexes(indexList, collection, {log: false});
eventHandler.on("done", function() {
      callback();
});
```

as the callback passed as argument (if it's passed at all) is only called on the listener *done*.

This last structure can be used to define personalized listeners when the events *dropIndex*, *createIndex*, *droppedIndex* and *createdIndex* occur. To do so, it's important to know the information "emitted" when these events are fired:

- **dropIndex, createIndex**: arguments:
      - *collection* 
      - *index*
      - *options* 
- **droppedIndex, createdIndex**: arguments:
      - *collection* 
      - *name* 
      - *index*
      - *options*
- **done**: arguments:
      *none*

The options emitted are the same as the argument passed to the function syncIndexes (the object with fields log and verbose).

As stated before, in case the name of the index to create is not provided, it's automatically generated by mongodb. That's the reason why such information is present in the arguments above: it's not necessarily part of the index during the creation.

Also, to personalise the messages, you can disable the logging in the options (which is true by default) and create your listener for the event desired:

```javascript
var eventHandler = syncIndexes(indexList, collection, {log: false}, callback);

eventHandler.on("createIndex", function(collection, index, options) {
      console.log(JSON.stringify(index.key) + " -> " + collection.s.name);
});
```

The result will be something like:

```
{"newKey":1} -> myCollectionName
{"newKey2":1} -> myCollectionName
```

However, if you want to display this message only after being sure the index was inserted in the collection, use the event *createdIndex* instead:

```javascript
var eventHandler = syncIndexes(indexList, collection, {log: false}, callback);

eventHandler.on("createdIndex", function(collection, name, index, options) {
      console.log("Index created has name " + name + ". It has the key " + JSON.stringify(index.key));
});
```

will give:

```
Index created has name personalizedName. It has the key {"newKey":1}
Index created has name newKey2_1. It has the key {"newKey2":1}
```

The events *dropIndex* and *droppedIndex* can be configured analogously.
