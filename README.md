# MongoDB Essential Training
This is the repository for the LinkedIn Learning course `MongoDB Essential Training`. The full course is available from [LinkedIn Learning][lil-course-url].
## Instructions

All instructions / commands used in the course are available in this repository for easy copy-paste access.

The datasets used in this course are available for download in the [datasets folder](/datasets/). Please download all three and import them into your database:

```sh
mongoimport --db=sample_data inventory.json
mongoimport --db=sample_data movies.json
mongoimport --db=sample_data orders.json
```

## MongoDB

### Mongo Installation

Follow the instructions in this tutorial [install-mongodb-on-ubuntu](https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-ubuntu/). Once that the db is installed, one can use the GUI for it, download and install [Compass](https://www.mongodb.com/products/compass).

> **NOTE**: Install the MongoDB Community Edition.

```bash
# install node version manager (optional)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash

# install mongo version manager
git clone git://github.com/aheckmann/m.git && cd m && make install
```

A proper dev environment requires ``mongod``, ``mongosh`` and ``mongodb-database-tools``.


### Manage MongoDB Service

```bash
# Start MongoDB
sudo systemctl start mongod

# Verify that MongoDB has started successfully
sudo systemctl status mongod

# Enable MongoDB from starting boot
sudo systemctl enable mongod

# Restart MongoDB
sudo systemctl restart mongod

# Begin using MongoDB
mongosh

# Run mongo GUI
mongodb-compass
```

### Mongod

Default settings

* port: 27017
* data storage: `/data/db` or `C:\data\db`

```bash
# create target dir for the database
mkdir mongod_only

# set the path for the db
mongod --dbpath mongo_only

# enter shell
mongosh

# basic commands
> show databases
```

### Replica Set in CLI

Redundant information in multiple instances, with recommendations

* at least 3 replica set members
* choose an uneven number of voting replica set members

Note: for production it is recommended to use X.509 certificates for the replica set.

```bash
# create dir for the set
mkdir replica_set_cmdline
cd replica_set_cmdline

# create a key file for the db
openssl rand -base64 755 > keyfile

# set the current user permissions
chmod 400 keyfile

# use shell parameter extension to create a set of dirs
mkdir -p m{1,2,3}/db

# configure the replica set for all set members
mongod --replSet myReplSet --dbpath ./m1/db --logpath ./m1/mongodb.log --port 27017 --fork --keyFile ./keyfile
mongod --replSet myReplSet --dbpath ./m2/db --logpath ./m2/mongodb.log --port 27018 --fork --keyFile ./keyfile
mongod --replSet myReplSet --dbpath ./m3/db --logpath ./m3/mongodb.log --port 27019 --fork --keyFile ./keyfile
```

To setup the replica set go into the mongo shell

```bash
# shell
mongosh

# start setup 
test > rs.initiate()

# switch for the db with configuration options
test > use admin

# create a user to setup the replica set
admin > db.createUser({user: 'UserName', pwd: passwordPrompt(), roles: ["root"]})

# configure a pass auth account
admin > db.getSiblingDB("admin").auth("UserName", passwordPromt())

# associated the other two replica members
admin > rs.add("localhost:27018")
admin > rs.add("localhost:27019")

# verify the replica set is working
admin > rs.status()

# or  use an alternative way to verify the replica set
admin > db.serverStatus()["repl"]
```

To remove the local replica set files

```bash
# kill the running instances
killall mongod

# go to parent dir
cd ..

# remove dir with replica set structure
rm -rif replica_set_cmdline
```

### Replica Set from Files

Replica set can be set with configuration files

```bash
# create dir for the set
mkdir replicaset
cd replicaset

# create a key file for the db
openssl rand -base64 755 > keyfile

# set the current user permissions
chmod 400 keyfile

# use shell parameter expansion to create a set of dirs
mkdir -p m{1,2,3}/db

# create a config file
touch m1.conf
```

The content of `m1.conf` has to be as follows

```bash
storage:
    dbPath: m1/db
net:
    bindIp: localhost
    port: 27017
security:
    authorization: enabled
    keyFile: keyfile
systemLog:
    destination: file
    path: m1/mongod.log
processManagement:
    fork: true
replication:
    replSetName: mongodb-essentials-rs
```

The other config files, `m2.conf` and `m2.conf`, follow the same structure.

```bash
# configure the replica set for all set members
mongod -f m1.conf
mongod -f m2.conf
mongod -f m3.conf
```

Then using the shell

```bash
# shell
mongosh

# switch for the db with configuration options
test > use admin

# create a variable to store configuration
admin > config = {
    _id: "mongodb-essential-rs",
    members: [
        { _id: 0, host: "localhost:27017" },
        { _id: 1, host: "localhost:27018" },
        { _id: 2, host: "localhost:27019" }
    ]
}

# set the replica set
rs.initiate(config)

# create a first user
db.createUser({user: 'FirstUser', pwd: passwordPrompt(), roles: ["root"]})

# authenticate against the created database
db.getSiblingDB("admin").auth("FirstUser")

# to get information from the replica set
rs.status()
db.serverStatus()['repl']
```

### Import Data Sets

Assuming that some json or csv files exist in your current path

```bash
cd datasets
mongoimport --username="FirstUser" --authenticationDatabase="admin" --db=sample_data inventory.json
mongoimport --username="FirstUser" --authenticationDatabase="admin" --db=sample_data movies.json
mongoimport --username="FirstUser" --authenticationDatabase="admin" --db=sample_data orders.json
```

### Debug installation

* Check in the `*.log` files
* Disable the fork option in the m1.conf file
* Check the Oplog with the command

```bash
admin > use local

local > db.oplog.rs.find({"o.msg": {$ne: "periodic noop"}}).sort({$natural: -1}).limit(1).pretty()
```

* Increase the log level 

```bash
# to read current log levels
local > db.getLogComponents()

# to set new levels
local > db.adminCommand({ setParameter: 1, logLevel: 2})
```

* Go to Stack Overflow

## Working with MongoDB

Basic operations

```bash
# set the right database
test > use blog

# show the tables in the db
show collections

# insert document into db blog
db.authors.insertOne({"name": "Naomi Pentrel"})
```

Inserting documents

```bash
db.authors.insertOne({"name": "Joe Nash"})
db.authors.insertMany([
    {"name": "Eliot Horowitz"},
    {"name": "Dwight Merriman"},
    {"name": "Kevin P. Ryan"}
])

# read a single document
db.authors.findOne()

# to read multiple documents
db.authors.find()

# filter documents
db.authors.find({ "name": "Naomi Pentrel"})
# or inside the mongosh one can remove the ""
db.authors.find({ name: "Naomi Pentrel"})

# update single document using $set operator
db.authors.updateOne(
    { name: "Naomi Pentrel"},
    { $set: { website: "https://naomi.codes" } }
)
db.authors.find({ name: "Naomi Pentrel"})

# update all documents in the collection
db.authors.updateMany(
    { },
    { $set: { books: [] } }
)
db.authors.find()

# delete a single document
db.authors.deleteOne({ name: "Joe Nash" })

# delete all documents in collection
db.authors.deleteMany({})
```

### Covered Query using indices

Create an index on common query patters only

* single field
* partial
* compound
* multi-key
* text
* wildcard
* geospatial
* hashed

```bash
db.authors.find()

# to create an index only based on the name field
db.createIndex({ name: 1 })
```

### Configuring Durability

```bash
db.authors.insertOne(
    {"name": "Joe Nash"},
    {
        w: "majority",    # default
        j: "true",        # all writes have to be persisted to disk
        wtimeout: 100     # prevents write ops to be locked
    }
)
```

``{w: 1, j: true}`` means that one replica set member must write to its journal before the write is acknowledged as having succeeded. If the replica set member temporarily becomes unavailable before the write can be replicated, the write could be rolled back.


### findOne and find

```bash
test > use sample_data
sample_data > show collections
sample_data > db.inventory.findOne()
sample_data > db.orders.findOne()
sample_data > db.movies.findOne()

# access nested keys
sample_data > db.movies.findOne({"ratings.mndb": 10})
{
    "title":"Match, The",
    "director":"Brock Jutson",
    "actors":["Krystyna Sapsford","Rosalynd Warboy","Jeremias Crisell"],
    "release_year":{"$date":"1997-06-01T16:39:10.000Z"},
    "genres":["Adventure", "Drama", "Horror", "Mystery", "Thriller"],
    "gross":44796132,
    "runtime_min":66,
    "ratings":{"soft_avocados":27,"mndb":10,"votes":1119}
}

# to read the index 0 in the genres array
sample_data > db.movies.findOne({"genres.0": "Musical"})
{
    "title":"The Adventures of Tom Thumb & Thumbelina",
    "director":"Niccolo Bris",
    "actors":["Lonni Fulger","Benedetto Jeandeau","Sylvester Argyle"],
    "release_year":{"$date":"1999-09-09T11:50:02.000Z"},
    "genres":["Musical"],
    "gross":1015338,
    "runtime_min":60,
    "ratings":{"soft_avocados":12,"mndb":7,"votes":2014}
}
```

The consistency and availability can be configured using the readConcern method.
This allows to see only data that is majority commited in all replicas.

Specifying a ``readConcern`` allows you to define the level of consistency you require when reading data. The default is ``local``, and you can change it to ``available``, ``majority``, or ``linearizable``.

Configurable readConcern in MongoDB by specifying:

* local (default)
* available (advanced - sharded clusters)
* majority
* linearizable (majority commited data only)

```bash
# set the concern
db.movies.findOne(<document>).readConcern("majority")
```

Specifying a ``readPreference`` allows you to read data from secondaries. The default is ``primary``, and you can change it to ``primaryPreferred``, ``secondary``, ``secondaryPreferred``, and ``nearest``.

Configurable readPreference in MongoDB by specifying:

* primary (default)
* priumaryPreferred
* secondary
* secondaryPreferred
* nearest (lowest latency)

### Comparison Operators

* $eq (=)
* $gt (>)
* $gte (>=)
* $lt (<)
* $lte (<=)
* $ne (!=)
* $in
* $nin

```bash
test > use sample_data
sample_data > db.inventory.findOne()
sample_data > db.inventory.findOne({ "variations.quantity": { $gte: 8 } })
{
    "_id":"782299567-7",
    "name":"Audi",
    "model":"A8",
    "year":2003,
    "price":6712.05,
    "source":"Yoveo",
    "sale_frequency":"Daily",
    "variations":[
        {"variation":"Turquoise","quantity":12},
        {"variation":"Teal","quantity":17},
        {"variation":"Pink","quantity":0},
        {"variation":"Yellow","quantity":21},
        {"variation":"Indigo","quantity":12}
    ]
}

sample_data > db.inventory.findOne({ "variations.quantity": { $gte: 17 } })
sample_data > db.inventory.findOne({ "variations.quantity": { $gte: 18 } })
sample_data > db.inventory.findOne({ "variations.quantity": { $gte: 22 } })
sample_data > db.inventory.findOne({ "price": { $lt: 1000 } })
sample_data > db.inventory.findOne({ "price": { $lt: 2000 } })
sample_data > db.inventory.findOne({ "price": { $lt: 1700 } })

sample_data > db.inventory.findOne({ "variations.variation": { $in: [ "Blue", "Red" ] } })
{
    "_id": "539527465-0",
    "name": "Oldsmobile",
    "model": "Bravada",
    "year": 1992,
    "price": 1966.35,
    "source": "Yoveo",
    "sale_frequency": "Yearly",
    "variations": [
        {
            "variation": "Red",
            "quantity": 16
        }
    ]
}

sample_data > db.inventory.findOne({ "variations.variation": { $nin: [ "Blue", "Red" ] } })
{
    "_id": "782299567-7",
    "name": "Audi",
    "model": "A8",
    "year": 2003,
    "price": 6712.05,
    "source": "Yoveo",
    "sale_frequency": "Daily",
    "variations": [
        {
            "variation": "Turquoise",
            "quantity": 12
        },
        {
            "variation": "Teal",
            "quantity": 17
        },
        {
            "variation": "Pink",
            "quantity": 0
        },
        {
            "variation": "Yellow",
            "quantity": 21
        },
        {
            "variation": "Indigo",
            "quantity": 12
        }
    ]
}
```

### Logical operators

 ```bash
use sample_data
db.inventory.findOne()

# array of conditions for AND
db.inventory.findOne(
    {
        $and: [
            { "variations.quantity": { $ne: 0} },           # cond 1
            { "variations.quantity": { $exists: true } }    # cond 2
        ]
    }
)

# array of conditions for OR
db.inventory.findOne(
    {
        $or: [
            { "variations.variation": "Blue" },
            { "variations.variation": "Green" }
            { "variations.variation": "Teal" },
        ]
    }
)

# price <= 8000 AND variation != Blue
db.inventory.findOne(
    {
        $nor: [
            { price: { $gt: 8000 } },
            { "variations.variation": "Blue" }
        ]
    }
)

# price must not be greter than 20000
db.inventory.findOne(
    {
        $not: { price: { $gt: 20000 } },
    }
)
 ```

### Sort, skip, limit

* When the sort is a common query pattern, use an index
* If you have no index, sort and limit is faster
* MongoDB always performs sort, skip, limit in that order

```bash
db.movies.find().sort({title: 1})

# sorting ascending
db.movies.find({}, { title: 1, genres: 1 }).sort( {title: 1})

# sorting descending
db.movies.find({}, { title: 1, genres: 1 }).sort( {title: -1})

# sorting with multiple fields
db.movies.find({}, { title: 1, genres: 1 }).sort( {director: 1, title: 1})

# skip the first 100 results
db.movies.find({}, { title: 1, genres: 1 }).sort( {title: 1}).skip(100)

# limit the number of results
db.movies.find({}, { title: 1, genres: 1 }).sort( {title: 1}).skip(100).limit(5)
```

### UpdateOne and UpdateMany

In ``db.collection.updateOne({ <document> }, { <document> })``, the first document specifies the match condition and the second document specifies how the matched documents are to be updated.

* ``{ $set: {msg: "Hello world!"} }``
* ``{ $unset: {msg: ""} }``
* ``{ $inc: {quantity: -1, ordered: 1} }``
* ``{ $mul: {price: 0.9} }``
* ``{ $max: {bid: 500} }``
* ``{ $min: {lowest_available_price: 500} }``

```bash
use blog
show collections
db.authors.find()
db.authors.updateOne(
    { name: "Naomi Pentrel" },                # <src document>
    { $set: { message: "Hello World!" } }     # changes
)
db.authors.find()

# update all documents adding the message field
db.authors.updateMany(
    {},
    { $set: { message: "Hello" } }
)
db.authors.find()

# update all documents removing the message field
db.authors.updateMany(
    {},
    { $unset: { message: "" } }
)
db.authors.find()
```

### Arrays

```bash
use sample_data
db.movies.findOne()

# get documents that include the genre Comedy
db.movies.find({genres: "Comedy"})

# get specific set of genres
db.movies.find({genres: ["Comedy", "Drama", "Thriller"]})

# get movies containing all items in the array
db.movies.find({genres: { $all: ["Comedy", "Drama"] } })

# get a docuiment that match the specified conditions
db.inventory.findOne()
db.inventory.find(
    {
        variations: {
            $elemMatch: {
                variation: "Blue",
                quantity: { $gt: 8  }
            }
        }
    }
)

# modify a document temporarily, changing the array
db.movies.findOne()
db.movies.updateOne(
    {
        title: 'The Adventures of Tom Thumb & Thumbelina'
    },
    {
        $push: { genres: "Test" }
    }
)
db.movies.findOne( {title: 'The Adventures of Tom Thumb & Thumbelina'} )

# 
db.movies.updateOne(
    {
        title: 'The Adventures of Tom Thumb & Thumbelina'
    },
    {
        $addToSet: { genres: "Test" }
    }
)
db.movies.findOne( {title: 'The Adventures of Tom Thumb & Thumbelina'} )
db.movies.updateOne(
    {
        title: 'The Adventures of Tom Thumb & Thumbelina'
    },
    {
        $addToSet: { genres: "Green" }
    }
)

# removes the last item in the genres array -> index 1
db.movies.findOne( {title: 'The Adventures of Tom Thumb & Thumbelina'} )
db.movies.updateOne(
    {
        title: 'The Adventures of Tom Thumb & Thumbelina'
    },
    {
        $pop: { genres: 1 }    # the first index in the arrya is -1
    }
)
db.movies.findOne( {title: 'The Adventures of Tom Thumb & Thumbelina'} )
db.movies.updateOne(
    {
        title: 'The Adventures of Tom Thumb & Thumbelina'
    },
    {
        $pop: { genres: 1 }
    }
)
db.movies.findOne( {title: 'The Adventures of Tom Thumb & Thumbelina'} )

```

### Transactions

To perform operations on multiple documents in an atomic way, particularly, when you must ensure your application never reads data while only some operations from a set have been performed.

* Guarantee atomicity of reads and writes to multiple documents
* reads return all documents as they are when the read begins
* either all writes happen or none do
* transactions can be used across multiple operations, documents, collections and databases

```bash
use blog

# create a session object
session = db.getMongo().startSession( { readPreference: { mode: "primary" } } )

# start transaction
session.startTransaction()

# perform operations on the blog database
session.getDatabase("blog").updateMany( {}, {$set: {message: "Transaction occured"} } )

# to finish the transaction
session.commitTransaction()

# to close the session
session.endSession()
db.authors.find()
```

Notes:

* Use transactions only when absolutely needed
* Overuse can lead to performance degradation
* if you need transactions a lot, refactor your the data model

### $exp

For more examples see [aggregation](https://www.mongodb.com/docs/manual/reference/operator/aggregation/) page.

```bash
use sample_data
db.movies.findOne()
db.movies.find({ title: 1, ratings: 1 })
{
    "title": "Jade",
    "ratings": {
        "soft_avocados": 57,
        "mndb": 3,
        "votes": 1020
    }
}
{
    "title": "She's the One",
    "ratings": {
        "soft_avocados": 47,
        "mndb": 3,
        "votes": 2518
    }
}

# in order to compare ratings in the same scale
db.movies.find(
    { $exp: 
        {   # looks for mndb * 10 > soft_avocados  
            $gt: [
                {$multiply: ["$ratings.mndb", 10]},  # $ points to the value of the field
                "$ratings.soft_avocados"
            ] 
        } 
    }
)
```

### example of app

```bash
use order_app
db.oders.insertMany([
    {
        "time": Date(),
        "items": [{
            "name": "Fries",
            "quantity": 1,
            "price": 2.99
        },{
            "name": "Diet Coke",
            "quantity": 1,
            "price": 1.99
        }],
        "delivery_cost": 3.50,
        "total": 8.48,
        "user": {
            "name": "Naomi Pentrel",
            "address": "Sample Street 123",
            "email": "naomi@email.com",
            "phone": "12345678910",
            "balance": 10
        },
        "restaurant": {
            "name": "Burger King",
            "address": "Somewhere in Amsterdam"
        }
    },
    {
        "time": Date(),
        "items": [{
            "name": "Fries",
            "quantity": 1,
            "price": 2.99
        },{
            "name": "Diet Coke",
            "quantity": 2,
            "price": 1.99
        }],
        "delivery_cost": 3.50,
        "total": 10.47,
        "user": {
            "name": "Naomi Pentrel",
            "address": "Sample Street 123",
            "email": "naomi@email.com",
            "phone": "12345678910",
            "balance": 10
        },
        "restaurant": {
            "name": "Burger King",
            "address": "Somewhere in Amsterdam"
        }
    }
])

db.orders.findOne()

# create indexes based on emain and restaurant name
db.orders.createIndex({"user.email": 1, time: 1})
db.orders.createIndex({"restaurant.name": 1, time: 1})

# query to find the most expensive order
db.orders.find().sort({total: -1}).limit(1)    # -1 means descending order

# sort orders by restaurant name
db.orders.find().sort({"restaurant.name": 1})

# return orders where the order total is bigger than the balance
db.orders.find({
    $expr: {
        $gt: [
            "$total",
            "$user.balance"
        ]
    }
})

# apply a 10% discount to all orders
db.orders.updateMany({}, { $mul: {total: 0.9} })
```

### Pipeline stages

See documentation in [aggregation-pipeline](https://www.mongodb.com/docs/v6.0/reference/operator/aggregation-pipeline/).

### $group

```bash
use sample_data
db.inventory.findOne()
{
    "_id": "539527465-0",
    "name": "Oldsmobile",
    "model": "Bravada",
    "year": 1992,
    "price": 1966.35,
    "source": "Yoveo",
    "sale_frequency": "Yearly",
    "variations": [
        {
            "variation": "Red",
            "quantity": 16
        }
    ]
}

# the aggregate method takes an array of stages as input
db.inventory.aggregate( [
    {
        # the group operator takes a document as input
        $group: {
            _id: "$source",
        }
    }
] )

# to get the number of counts frome each source
db.inventory.aggregate( [
    {
        $group: {
            _id: "$source",
            count: { $sum: 1 }        # created count field
        }
    }
] )

# adds a field for the items and push the name of each car to items field (as an array)
db.inventory.aggregate( [
    {
        $group: {
            _id: "$source",
            count: { $sum: 1 },
            items: { $push: "$name" }
        }
    }
] )

# adds an average price field and calculate the average price for all the fields in that group
db.inventory.aggregate( [
    {
        $group: {
            _id: "$source",
            count: { $sum: 1 },
            items: { $push: "$name" },
            avg_price: { $avg: "$price"}
        }
    }
] )
```

### $bucket

```bash
db.inventory.aggregate( [
    {
        $bucket: {
            groupBy: "$year",
            boundaries: [1980, 1990, 2000, 2010, 2020],
            default: "Other"        # items outside the boundaries
        }
    }
] )

# add fields into the buckets
db.inventory.aggregate( [
    {
        $bucket: {
            groupBy: "$year",
            boundaries: [1980, 1990, 2000, 2010, 2020],
            default: "Other",
            output: {
                count: { $sum: 1 }
                cars: { $push:
                    {name: "$name", model: "$model"}
                }
            }
        }
    }
] )

# to create buckets with equaly sized groups
db.inventory.aggregate( [
    {
        $bucketAuto: {
            groupBy: "$year",
            buckets: 5
        }
    }
] )
```

### $unwind

If you have a document with an array you can use the $unwind stage to create one output document for each array element.

```bash
db.inventory.findOne()
{
    "_id": "400520522-4",
    "name": "Jaguar",
    "model": "XJ Series",
    "year": 1996,
    "price": 12992.62,
    "source": "Vinder",
    "sale_frequency": "Often",
    "variations": [
        {
            "variation": "Pink",
            "quantity": 7
        },
        {
            "variation": "Violet",
            "quantity": 27
        }
    ]
}

# to create a document for each car variation
db.inventory.aggregate( [
    {
        $unwind: "$variations"
    }
] )
db.inventory.findOne()

# the match state targets a paticular value
db.inventory.aggregate( [
    {
        $unwind: "$variations",
    },
    {
        $match: { "variations.variation": "Purple" }
    }
] )

# to pre-filter the documents before the unwind
db.inventory.aggregate( [
    {
        $match: { "variations.variation": "Purple" }
    },
    {
        $unwind: "$variations",
    },
    {
        $match: { "variations.variation": "Purple" }
    }
] )
```

### $out

The dollar out stage allows you to store the results of an aggregation pipeline in a new collection.

```bash
# creates a new collection in database from pipeline
db.inventory.aggregate( [
    {
        $match: { "variations.variation": "Purple" }
    },
    {
        $unwind: "$variations",
    },
    {
        $match: { "variations.variation": "Purple" }
    },
    {
        $out: { db: "sample_data", coll: "purple" }
    }
] )
show collections
db.purple.find()

# allows to add results into an existing collection
db.inventory.aggregate( [
    {
        $match: { "variations.variation": "Purple" }
    },
    {
        $unwind: "$variations",
    },
    {
        $match: { "variations.variation": "Purple" }
    },
    {
        $merge: {
            into: "purple",
            on: "_id",
            whenMatched: "keepExisting"
            whenNotMatched: "insert"
        }
    }
] )
```

### $function

The dollar function pipeline operator allows you to write JavaScript functions that operate on the field values in your documents.

```bash
db.movies.findOne()
{
    "title": "Perez.",
    "director": "Thorsten Showl",
    "actors": [
        "Shirlee Skippon",
        "Rozina Noquet",
        "Meier Gresley"
    ],
    "release_year": {
        "$date": "1991-04-27T13:56:08.000Z"
    },
    "genres": [
        "Action",
        "Adventure",
        "Animation",
        "Sci-Fi"
    ],
    "gross": 34511163,
    "runtime_min": 221,
    "ratings": {
        "soft_avocados": 65,
        "mndb": 2,
        "votes": 2707
    }
}

# implements a sorting of actors aphabetically
db.movies.aggregate([
    {
        $project: {        # to change or project away certain fields
            title: 1,
            actors: {
                $function: {
                    body: 'function(actors) { return actors.sort(); }',
                    args: [ "$actors" ],
                    lang: "js"
                }
            }
        }
    }
])
```

### lookup

The $lookup stage allows you to perform joins, that is to pull in data from another collection based on a matching field. This requires

* create and index on the foreignField
* common query patters should rarely require joins

```bash
db.orders.aggregate([
    { $lookup: {
        from: "inventory",
        localField: "car_id",
        foreignField: "_id",
        as: "car_id"
    } }
])
```

### Perfomance

A native profiler can be active to collect data under a specified limit ``db.setProfilingLevel(1, {slowms: 20})``, such data can be query ``db["system.profile"].find()``.

```bash
db.movies.explain("executionStats").aggregate( [
     { $project: {
          release_year: {$year: "$release_year"},
          title: 1
     } },
     { $lookup: {
          from: "inventory",
          localField: "release_year",
          foreignField: "year",
          as: "year"
     } }
] )
```

Common optimizations

* $sort + $limit in that order
* $project as the final stage
* hinting: ``db.collection.aggregate(pipelin, {hint: "index_name"})``
* Alanytics nodes
* kill bad performace operations

```bash
db.currentOp(true)
db.adminCommand({ "killOp": 1, "op": OP_NUMBER })
```

### Aggregation Pipelines

An aggregation pipeline consists of one or more stages, where each stage allows you to manipulate data, transform data, and make use of a broad range of aggregation operators.

* find out ho many movies were released in each genre since 2000
* store the result in a new collection called generes2000

```bash
db.movies.findOne()
{
    "title": "Magnificent Bodyguards (Fei du juan yun shan)",
    "director": "Michelle MacNeice",
    "actors": [
        "Renaldo Millgate",
        "Aldwin Lanston",
        "Frannie Baxendale"
    ],
    "release_year": {
        "$date": "2014-06-14T10:28:31.000Z"
    },
    "genres": [
        "Animation",
        "Drama",
        "Fantasy"
    ],
    "gross": 108518586,
    "runtime_min": 60,
    "ratings": {
        "soft_avocados": 9,
        "mndb": 1,
        "votes": 5854
    }
}

db.mock_movie_data.aggregate([
    # stage to set all fields to be used
    { $project: {
        release_year: { $year: "$release_year" },    # grabs year from field
        genres: 1,
        runtime_min: 1,
        title: 1
    } },
    { $match: { release_year: { $gte: 2000 } } },    # filter by the year of itnerest
    { $unwind: "$genres"},         # create single documents per each genre
    { $group: {                    # group by genre
        _id: {
            genres: "$genres",
        },
        movies: {
            $push: { 
                title: "$title", 
                runtime_min: "$runtime_min", 
                release_year: "$release_year"
            } 
        },
        count: { $sum: 1 }        # adds 1 per each document found
    } },
    { $out: { db: "sample_data", coll: "genres2000" } }
])
```

### Relational vs document

1. Storing JSON documents
    * Supported by both
    * Relational databases store JSON in blobs
    * Relational - good performance but not built for it
    * Document - optimal performance
1. Storing tables
    * Relational models are optimized for working with tables and joins
    * MongoDB is not optimized for working with tables and joins primarily
1. Storing data in the data models best for each database
    * you can query MongoDB with SQL
    * performance, ease of development, learning time, scale 

### Data Modeling

One-to-one relationships: store them together unless

* the document gets too big
* you rarely use the information

One-to-many relationships

* one to few: store both in one collection using nesting or arrays
* one to many: store them separatetly with links

Many-to-many relationship

* store links

### Schema

MongoDB supports schema validation and enforces a schema for a collection if you configure it to do so. By default, MongoDB is schemaless and does not enforce any structure on your data, aside from requiring that each object must have an _id field. If no _id field is provided, MongoDB will add one of you.

```bash
# add validator for movies
db.runCommand(
    {
        collMod: "movies",
        validator: {
            $jsonSchema{
                bsonType: "object",
                required: ["title", "director"],
                properties: {
                    title: {bsonType: "string"},
                    director: {bsonType: "string"},
                }
            }
        },
        validationLevel: "moderate"        # applied only to new documents
    }
)

# failing insert with unvalid schema
db.movies.insert({"title": "test"})     # document fail validation
```


[lil-course-url]: [https://www.linkedin.com/learning/](https://www.linkedin.com/learning/mongodb-essential-training)
[lil-thumbnail-url]: http://

