Realm Training Notes
====================

These notes reflect my (Nate Contino's) personal interpretation of Ian
Ward's Realm Training presentations from February 2020. Any
inaccuracies are probably the result of my own misinterpretation, or
possibly the result of the necessary simplification Ian used to present
the result of hundreds of thousands of hours of programming in a single
week. Feel free to contact me directly if you have any questions or
believe a section should be changed.

Chapter 1: Realm Sales Pitch
============================

- People love mobile. Companies feel like they need a mobile presence,
  and that presence *must* be a positive one. Essentials for a positive
  mobile presence:
  - reactive, responsive UI
  - functionality while online & while offline: it is frustrating when
    an app loses state/refuses to work just because you're on the
    subway, in the mountains, in your basement, etc.
  - realtime collaboration: don't want to wait 5 minutes for an order/
    post/comment/image to show up (if at all). Especially important in
    social apps, but relevant for any app where a client produces
    content and pushes it up to a server.
- How do devs make this happen today? By spending effort on...
  - custom code/logic to make app interaction reliable and resilient
    despite network conditions, server wonkiness, all manner of fail
    cases
  - heavy use of asynchronous patterns
  - SQLite + ORM + REST + ...
  - mobile/server developer coordination to keep schemas consistent
    across languages/implementations, produce new endpoints, validation,
    security
- mobile is a "hostile environment": variable latency, connections lost,
  forced shutdowns (by user or OS or power loss)
- many different environments to support: watch, tablet, phone, car,
  laptop
- backend: wide array of microservices, REST/other APIs, DBs, old
  systems to support at any org
- stitching together different mobile platforms/backend fragmentation
  causes a lot of headaches. Validation, schemas, request timeouts,
  security, and edge cases for all of these can take up most of an org's
  dev time
- Creating new apps/services/features bogged down by maintenance and
  overhead of all of this coordination
- devs need fast, adaptive APIs; SDK support for many environments;
  thorough, up-to-date documentation; ample learning material;
  reliability

Realm vs. Status Quo
--------------------

- Popular solution: REST APIs
  - passive "give me this page" use case works great
  - interactive use case "give me this thing, I'll edit it, send you
    changes" or collaboration ... quickly becomes very complicated
  - conflicts due to network partitions/failed calls
  - schema validation, encoding/decoding are all highly manual and
    error prone
  - lots of complex exception conditions for which developers need to
    write custom code
     - time, effort, $$$ spent on this instead of new features or
       improvements
- GraphQL solves some of these issues, but not all: still have to handle
  network conditions manually, still unidirectional, still have to
  resolve conflicts yourself, still have to encode/decode everywhere

A "Live Object" Database
------------------------

- Objects defined in your application are the schema for your database
- Memory mapping used so that the "object" you see in your application
  is really just a pointer to that object in the database
  - Because your in-memory object is just a pointer, every access points
    to the most recent version of your object
- notification API to update the UI without polling for changes

Realm Object Server (ROS)
-------------------------

- real-time bidirectional sync between connected clients using the
  server as a common point of contact
- automatic conflict resolution across devices so every connected device
  ends up with the same data
- support for schema changes

Realm Mobile DB
---------------

- offline first: you write to your local DB instance, and a separate
  thread queues up your change and syncs it to the server when
  a connection is available (could be immediate, could be minutes, could
  be days)
- change notifications, subscribe to find out when a query's results
  change
- no manual network access: all is abstracted behind database
  interaction

Realm Sync
----------

- sync on write
- "strongly eventually consistent"
- efficient data transfer: compressed, **field-based diffs**
- full & query-based[with caveats] sync
- can sync up to 12 realms at a time (limitation based on how many
  files Android/iOS allow you to open at a time)

Realm: A New Paradigm
---------------------

- no custom network handling
- schema defined in code
  - create an object, store in realm with realm.write()
- objects are live
- Some sample use cases:
  - chefs/stock employees track inventory, even offline in a freezer,
    which syncs to a central DB that orders more/halts future order
  - cruises: track when customers use services (food, drink, jet ski
    rental) and charge them when those services are synced to a central
    DB. Islands/ocean aren't known for the best data connection
  - airline: checklists/heartbeats during flight tracking altitude,
    heading, fuel, notes etc. synced after landing. Central DB runs
    aggregations and can notice macro-problems, anticipate equipment
    issues
  - health care provider/bank with many separate APIs: ping all changes
    from each of those APIs into a central view so an app just hits one
    API which doesn't rely on slower/less performant APIs, instead just
    pulls from central view

Questions/Thoughts from Customer Sell:
--------------------------------------
1) "Realm handles it": how much does realm actually handle, how much do
   I have to think about?
2) Use cases that accumulate data infinitely (IoT, for instance): how to
   handle? Migrate data out manually with a TTL/LRU cache type system?
3) Keeping schema in sync between clients: if I have Android, .NET,
   JS, and iOS clients, how do I roll out consistent schema changes to
   all of those? What is the best practice?
4) What about web apps? Is there any way to integrate realm with those?
   (realm JS sdk uses realm files to store DB, just like other clients
   -- normally this isn't a capability we assume in browser)

Chapter 2: Technical Overview
=============================

What is a Realm?
----------------

- a Realm persistently stores objects whose schemas are defined by
  Realm Objects
- Realm vs. MongoDB:
  - Realm == database
  - each Realm is comprised of heterogenous objects, just as a mongoDB
    database is comprised of collections, 
  - Object definition/schema in a Realm == collection (each Object
    defines a schema for a homogeneous group of objects, i.e. a Dog with
    a name and an age)
  - instance of an object == document in a collection (a dog named
    "Fudge" of age 14)
- Realm vs. general SQL databases
  - Realm == entire db
  - Object definition/schema in Realm == table
  - instance of an object == row in a table
- each Realm lives in a Realm file
- Object schemas can define embedded lists, and relations (foreign key)
- Realms are cross platform: can sync a Realm file between iOS, Android,
  .NET, and JS. Just need to define the object schemas in that
  environment's language
- ACID compliant
- no Realm Query Language: just uses the idiomatic query language for
  each client platform.
  - sample query for a list of brown doggos ordered by name:
    Realm.objects(Dog.self).filter("thing = 'brown'").sorted("name")
- can AES encrypt on device, store keys in device keystore for security

Sync
----
- for developers, interaction is the same as a local Realm (config is
  only difference)
- introduces public vs private realm distinction
- introduces "~" character, which expands into a unique user ID when
  used in a Realm path with a synced Realm
  - for instance: "~/settings" will expand to
    "my-unique-user-id/settings"
  - users automatically have write/admin permissions for private realms
    that begin with their path

Public Realms
-------------
- visible to all
- read-only for non-admins
- for instance: "/productCatalog" or "/announcements"

Private Realms
--------------
- per-user:
  - "/userA/shoppingCart"
  - only owner can read/write/administrate
- shared:
  - "/userA/wishList"
  - owner can grant read/write access to other users
  - ownly owner is admin
- no roles for now: users granted permissions individually based on ID

Server-side Integrations
------------------------
- Realm Object Server contains all synced Realms, all clients send
  all changes to ROS
- you can use .NET or JS SDKs OR GraphQL with any language to add custom
  logic to an instance of ROS
- listen/react to changes, administrate, validate, compliance, etc.
- can use SDK to get notifications for specific Realms/queries, OR...
- Global Listener: get a stream of all changes from all Realms (public
  or private) on server
- server side encryption, store secrets in k8s

Patterns
--------
1) Fetcher:
   - user creates FetchRequest.request ("get txns from bank account
     4284584") in a Realm
   - FetchRequest syncs up to the server
   - global/query notifier sends write to JS/.NET/GraphQL service, which
   - queries external API "GET /txns/4284584"
   - write API return value to the FetchRequest.return
   - ROS syncs returns down the client. Return value appears on screen/
     via notification depending on the expected API runtime
2) Poller:
   - user creates PollInterval to poll some external API in a Realm
   - global/query notifier sends write to JS/.NET/GraphQL service, which
     starts polling an external service for changes
   - when the service finds a change, writes data into ROS
   - client deletes PollInterval, global/query notifier sends to
     service, which stops polling
3) Dequeuer:
   - data pushed to JS/.NET/GraphQL service using Kafka, RabbitMQ, or
     some other message queue
   - JS/.NET/GraphQL service pushes that data into public, private, etc.
     Realms
   - when client syncs, new data shows up on their device

Realm Data Adapters
-------------------
- data synced between Realm Object Server and Some Other Database
- ROS sends changes to Some Other Database
- generally pushes data FROM realm INTO Some Other Database
- MongoDB-Realm Adapter based on this model, with custom
  bidirectional logic (pushing data from MongoDB to Realm generally
  overwrites data in Realm)
- Some Other Database used to include Postgres, MySQL and others but all
  of those were deprecated when MongoDB acquired Realm
- Adapter interface still exists so someone could write custom logic for
  any DB

Chapter 3: Technical Deep-Dive into Realm DB
============================================

Realm Core Database
-------------------
- focus on computing in a constrained environment (CPU, memory, storage)
  - this optimization leads to speed/efficiency
- Realm uses column-based storage
  - popular choice: row-based storage, where each object is saved as a
    struct in memory
    - inevitably wastes space for padding/alignment
    - reading is slower because you have to walk through a lot of data
      that you don't care about for non-indexed queries
    - spatial locality difficult because so much extra data in search
  - instead, realm objects are laid out in arrays of individual columns
    of at most 1k elements
  - searching through these columns can exploit vector operations in
    CPU, very fast
  - consequentially, writes are slower: need to write to multiple
    locations
  - column blocks (clusters) organized as a "tree of trees"
- data structure: B+ trees
  - self-balancing, log(n) lookups, wide branch factor
- table: B-trees of clusters
- cluster: column-oriented representation of a range of object
- ZigZag searching: uses "narrowest" search term first to limit the
  number of documents touched
- if a column in a cluster only covers a small, low-cardinality range,
  basically stores data in a Huffman tree
- concurrent access: Multiple Version Concurrency Control (MVCC):
  - only a single writer, which can only write on the latest version
  - snapshots preferred over locks (locks degrade performance)
  - tombstones (Postgres' choice) too expensive in mobile environment

Persistent Data Structures
--------------------------
- don't rewrite data to the same place: write new data somewhere else!
- change pointers between clusters in the B+ tree instead, GC old stuff
- memory-map clusters as read only (don't trust users not to write
  accidentally)
- make a change, new changes are staged and written
- operating system can handle paging more intelligently than we can
- need to close realm so that transactions end. Otherwise will build
  up a huge realm interim file (because open realms keep a list of
  current in-use snapshots)
- read transactions and write transactions:
  - realm.open() starts an implicit read transaction
  - realm.close() finishes that read transaction
  - writes in between are handled by explicit write transactions
  - write transaction starts/commits implicitly advance the enclosing
    read transaction to the most current version of the db
- realm.compact(): full "defrag" to organize the tree optimally (memory
  allocation results in gaps/holes if left unchecked). Need to do this
  occasionally if you want size to stay stable/optimal
  - can "automatically" compact with compactOnLaunch setting
- ACID: Only write data to new locations.
  - a header in the realm file contains three values:
    - pointer1, a pointer to the head of a B+ tree
    - pointer2, a pointer to the head of another B+ tree
    - which, which indicates which of pointer1 and pointer2 is the
      current valid head of the tree
  - write transaction commits overwrite the pointer that isn't the
    current head of the tree, then update the "which" value to point at
    the new head. Atomic update. Tree remains valid even if the
    operation is interrupted, battery dies, process killed, etc.
- Realm relies heavily upon memory mapping. On mobile, memory mapping
  is limited: both the number of regions mapped and the total size of
  each region. Can cause problems for realm devs because we have to fit
  the lowest common denominator of device, hardware differs in limits
  and bugs and is largely undocumented.

Zero-Copy Approach
------------------
- most DBs copy in app memory into DB memory into storage for writes
  and vice versa for reads (ORM)
- realm does NOT build a representation of your object's data into
  app memory!
- data accessors/mutators use C++ accessors to get data from memory
  mapped region
- SDK makes these look native (Dog.getAge()) but the data doesn't live
  on the stack or in the heap of your app.
- "The ultimate lazy loading"

Transactions
------------
- transaction object selects most-recent version of the Realm when
  created
- every accessor returns that version's data
- transaction can "move in time" to look at newer data
- Continuous Transaction: back live objects, main runloop increments
  version of Realm the transaction uses
- objects stable in each runloop iteration, but change between cycles
- so if you read at the beginning of a function, do some computation,
  then read again at the end of the function, data can't "change under
  you". Could change between function calls though.
- transactions are the only part of Realm that modify state
- transactions enable observable realms, auto-updating queries,
  query notifications

Thread Safety
-------------
- Accessors are NOT thread-safe
- can't pass objects across threads... but CAN destroy from any thread
- frozen objects: read-only, locked to a version of the Realm (don't
  update) but can be passed across threads

Chapter 4: Technical Deep-Dive into Realm Sync
==============================================
- Realm Sync: maintains the state of data in a realm across multiple
  devices -- client(s) and server
- "strongly eventually consistent": all replicas are guaranteed to
  converge on a single result if connected
- state is maintained with two parts:
  - change history: approximately equivalent to MongoDB's oplog. A
    stream of idempotent operations/transformations to your data. This
    history is generated using "operational transform", a means of
    boiling down user actions in a Realm into statements to which we can
    apply our merge rules.
  - merge rules: same in every client and in the ROS, control how we
    resolve conflicts in the change history.
- CAP (Consistency, Availability, Partition tolerance): databases can
  only pick three. Realm picks AP (Availability & Partition tolerance)
  - (why not consistent? Since clients sporadically go offline, querying
    a random selection of clients at any given time is going to give you
    vastly different results. But once they sync, they'll end up the
    same!)
- uses websockets over which we send compressed field-level data diffs
  - full-duplex: bidirectional
  - fast resume (for spotty networks)
  - interruption resistant
- ~20,000 concurrent clients as a GENERAL GUIDELINE
  - readers are cheap
  - writers are expensive (for all syncing clients and server!)
  - performance is highly dependent upon network conditions, client
    availability
- nor more than 10k objects/1MB per transaction for best performance
- remember that non-synced realms are already size sensitive: they
  should fit in any client's memory and storage. Syncing adds one
  additional constraint: all data has to be synced over the network!
  No more than 1GB as a GENERAL GUIDELINE
- synced realms are associated with sync workers
- sync does not allow destructive schema changes (removing a field or
  object)
- cannot convert a non-synced realm to a synced realm (because
  non-synced realms don't keep a history!)
- sacrifice customizability of protocol to abstract away distributed
  systems problems from mobile devs and prevent customers from shooting
  themselves in the foot
- because nodes are so sporadically connected, difficult/impossible to
  maintain invariants ("never let the bank balance go negative")
- because the protocol, merge rules, and history need to be the same
  on all nodes in the network to converge on states, all nodes need to
  upgrade before any nodes can use any protocol extensions

Conflict
--------
- A condensed version of the conflict resolution (merge) rules (there
  are over a hundred in real-world Realm):
  1) deletes always win
  2) last update wins
  3) list insertions order by time
  4) primary key is id: merge objects with same key
  In this case, "win" means "applied last"
  - these rules can get pretty complicated and wibbly wobbly timey
    wimey when you also account for *causality*: the notion that one
    event can influence a later event (you see someone added
    "chili powder" to the shopping list, this jogs your memory and you
    change "ground beef" to "ground turkey"). 
  - keep in mind that clients aren't always online, so one client
    might sync after 1 week of being offline, another client might
    sync after 1 day of being offline. Especially when editing lists,
    it's easy to see how we can end up with 100+ rules!
  - causality inferred based on version numbers changes are based upon
- so essentially each client keeps track of a change history and sends
  that history to the ROS. The ROS shares that history with all other
  clients. Because all of these "nodes" (server and client) have the
  same rules regardless of platform, each node ends up in the same
  state if it has the same changes.

Protocol
--------
- mobile device connections are unreliable, even in "good" network
  conditions!
- minimize latency: upload client changes ASAP
- prioritize sending chunks/small pieces instead of "big bang" (remember
  that it's highly likely you'll be interrupted!)
- no user thread blocking
- compact integer compression, string interning to keep chunks as small
  as possible (...but only compress > 1.5KB or some other low threshold
  where compressing isn't worth the effort)

Sync Storage
------------
- 1 byte of state translates to 2-4 bytes of instructions to get to that
  state (this can cause realm files to become large for clients that
  do not sync often!)
- fully sync'd client: zero storage overhead for history
- server must store history indefinitely
- realm file never shrinks except during a compact() event

Query-Based Sync
----------------
- DON'T USE IT
- create a subscription (w/TTL & name), get all objects newly changed
  that meet criteria
- can query Realm.subscriptions("<name>") to update/delete subscription
- update:true just like objects in Realm to update subscription
- this creates a server-side "partial realm"
- each partial realm written to disk
- every query evaluated for changes to every partial realm
- very expensive/non-scalable becaues every client's changes have to be
  evaluated for every active partial realm for every client
- list changes in particular can get very expensive very quickly

GraphQL
-------
- query language that lets you describe the return values based on a set
  schema of queryable relational objects
- 1 API endpoint, many resp/return pairings instead of REST where you
  have many endpoints each with a distinct resp/return pairing
- operations:
  - query (read)
  - mutation
  - subscription (changestream)
- ROS has endpoint per realm: /graphql/<realm-path>
- Realm pluralizes queries for readability (person -> people)
  (octopus -> octopuses) (this is customizable if you prefer octopi)
- singular & plural createObject(s) nodes
  - update policy: like SDKs, NEVER/ALL/MODIFIED
  - in general should use NEVER for creates, MODIFIED for updates
    - ALL is like MODIFIED but more expensive
- subscriptions via GraphQL give ALL matches on each tick, not just diff
  - I add one Student to a Class... your query for all Students gets the
    whole List<Student>, not just the new one!
- GraphiQL: GraphQL standard web-based query tool. Build your requests
  in browser (Compass->MQL is like GraphiQL->GraphQL)
  - autocomplete, schema introspection
  - open from Realm Studio with endpoint for your Realm
  - can open for browser, just need auth header and url
    - cloud-url/graphql/explore-<realm>
- don't send empty objects to current GraphQL implementation (will
  break)

Permissions
-----------
- old Realm manages these via hidden Realms
- user admin permissions in "~" personal path
- shared realms allow users to grant read/write permissions to others

Chapter 5: Realm Object Server (ROS)
====================================

Authentication
--------------
- JWT preferred, no revocation
  - initial sign in with JWT, after that via realm cloud token
- self-hosted ROS gives you the ability to write your own
  auth.authProvider to handle exactly the way you want
- POST to server with credentials & token: if valid, get Realm token
- issue new tokens until session expires or logout
- after auth, inits Realm Sync connection via GET to ROS root. We HTTP
  upgrade this connection to establish websocket.

Chapter 6: Realm Tools
======================
- utilities to tweak/analyze the ROS

`realm-stat`: profile stats of node state and history (oplog)
- can tell you if you need to compact, or if clients are doing
  suboptimal things, like overwriting unchanged fields in an object
  (doesn't hurt the server or client state, but generates unnecessary
  history)

`realm-hist`: history version info

`realm-vacuum`: compact & trim history simultaneously
- manual compact & trim is per-realm: realm-vacuum randomly selects 1 of
  the top 10 largest & least-recently vacuumed clients to vacuum.
- use `--ignore-clients` to force it, otherwise ROS might try to hold onto
  history in case an unsynced-for-a-long-time client tries to connect
  (specifically, if such a client has *tried* to connect, but failed to
  download the entire history. Basically ROS is an optimist.)

Chapter 7: Realm Cloud
======================
- Kubernetes
- AWS only, one cluster per region (us east, us west, eu west, eu east,
  ap southeast)
- Grafana, Kibana, Github all tied into the implementation
- A number of existing clients on Realm Cloud today; future on Stitch

Chapter 8: .NET SDK
===================
- Xamarin or Server

Data Models
-----------
- define your schema using normal primitives
- most values are nullable by default
- no inheritance hierachy allowed (can only inherit from RealmObject)
- no need for an explicit foreign key: just declare an I<Object> field
- break a foreign relationship with = null
- backlinks allow you to follow relationships in a backwards direction
  - IQueryable works well for this since you probably have to define a
    query for the back object
- native .NET binding does two-way binding to push/pull updates, very
  little boilerplate needed

Connecting
----------
- initial account creation/instantiation of Realm: connect with
  getInstanceAsync()
- subsequent connections: getInstance() will connect without hanging if
  you're offline... but if you're online, it can cause content to "jump"
  into the UI as the client syncs updates down.
- best practice: timeout whenever you use async in case a network
  connection can't be made
  - but keep in mind that a sync might require a large-ish download, so
    don't set your timeout too low. Give the client a chance to sync!

Writes
------
- must occur in a write transaction
  - write transactions have a lot of overhead, so limit them (batches)
  - writes can and do fail: handle errors!
    - but most of those errors are non-recoverable, so log/try again
- beginWrite() gives you a write transaction
  - "dispose" dotnet pattern: "using"
- if you don't call commit(), the write WILL be discarded
- Realm.Add() to write object to disk within a write transaction
- only one write at a time (blocking), so commit ASAP! Others may be
  waiting
- update existing objects using Add() with [update: true] & a primarykey
- no cascading deletes
- writes are always synchronous, so use a background thread
  - EXCEPT when write is a user action: then you can write in main
    thread for speed/responsiveness
- write commit on UI thread will advance read transactions on that
  thread to the most recent version of the realm

Refreshes
---------
- read transactions advanced to most recent version of realm:
  - between runloop iterations
  - when Refresh() is called
  - after a write Transaction.commit()

Results
-------
- IQueryable<type> collection returned from queries
- results computed on request, chaining very efficient as a result

Limitations
-----------
- Xamarin: certain platforms restrict total size of open files, memory
  mapping used in Realm tech

Chapter 9: Android SDK
======================
- Java or Kotlin
- many different devices, many different bugs
- main thread: 16ms per frame. I/O, networking discouraged
- MVVM (Model-View-ViewModel)
- Repository pattern (recent google-recommended practice):

```
Activity
   |
ViewModel (business logic)
   |
Repository (creating/fetching data)
[sqlite, cache, REST, etc. beneath here]
```

- Realm fits nicely into this pattern as the repository for any app

Data Models
-----------
- extend RealmObject
- @PrimaryKey for PK, @Index for index
- @LinkingObjects for inverse relationship ("backlink")
- @Required to make things non-nullable
- most objects are reactive, (you can add a change listener) including:
  - Realm
  - RealmResults
  - RealmObject
  - RealmList
- objects not thread safe, realms not thread safe

Connecting
----------
- create credentials (createUser option will make account, if none)
- create SyncUser
- create config (local: RealmConfiguration, sync: SyncConfiguration)
  - sync
  - initialData
  - timeout
- create realm with Realm.getInstance(config)
  - can override onSuccess() and onError() methods for success/error
    "notifications"
- can get/set default local config for ease of creation elsewhere in app
- can override connectionChange, downloadProgress, uploadProgress
  listeners of SyncSession (obtained via SyncManager.getSession(config))
- syncManager lets you refreshConnections() to force a reconnect attempt
  - can add a custom request header, auth header
- conf.maxNumberOfActiveVersions lets you tweak the max number of
  versions Realm keeps around, lower == lower memory overhead, potential
  issues when you try a lot of writes/use frozen objects

Queries
-------
- when you query, the WHOLE query is executed when you findAll(). So if
  you want to chain multiple conditions, you should do something like
  this:

  val res = realm.where<Project>()
               .equalTo("Tasks.name", "buy groceries")
               .findAll()
               .where()
               .equalTo("Tasks.isDone", true)
               .findAll();
  because query builder defaults to OR, not AND.
- android query language is very limited compared to iOS or even dotnet.
- change listeners get a changelist, an insertionlist, and a
  deletionlist
- RecyclerViewAdapter and RealmBaseAdapter provided to ease hooking up
  a query to a view (io.realm.android-adapters)

Writes
------
- write transaction with executeTransactionAsync() (lambda)
- can override transaction onSuccess() and onError() methods for success
  and error "notifications"
- get object with realm.createObject()
- write with copyToRealm<Type>()
- update with copyToRealmOrUpdate() with primary key

Refreshes
---------
- realm version updates to most recent on write transaction start & end
  - check your invariants IN the write transaction itself to be sure the
    object hasn't been updated by another write!
- can't pass managed object into async transaction because it's handled
  by a different thread... query again for your object!

Results
-------
- RealmResults are live, auto-updating
- interface similar to an ArrayList

RxJava2
-------
- support flowable & observables

Limitations
-----------
- app sandbox is secure unless rooted
- keystore only safe is secured with PIN/finderprint/password
- 32-bit devices can run out of memory at 300-400MB realms
- frozen objects keep a realm version around until they're deleted, can
  balloon memory
- android studio debugging, because of the way Realm overrides accessors
  android studio WILL lie to you about field values in the debugger.

Chapter 10: iOS SDK
===================
- Swift or Objective-C

Data Models
-----------
- need "@objc dynamic" annotation to use realm accessor/mutators
- no multi-class containers, query for multiple classes, or polymorphic
  casting
- link with Object & List properties
- inverse link with LinkingObjects
- RealmCollection defines interface

Connecting
----------
- get user
- set/get sync config
- instantiate session
- initial realm connection can fail, so try/catch it. Later accesses
  should always work
- create your realm objects with initializer, dictionary, or arrays
- append method, List behaves like normal array in swift

Queries
-------
- queries are predicate-based query language
- query execution is deferred until results are actually used
- notifications use reference token; store on class to continue getting
  updates
- notifications always delivered to thread that created notification
- notification handlers called asynchronously after write transaction
  commit
- write transaction notifications are coalesced
- thread safe reference: should resolve ASAP, pins version of realm
  until deallocated. Should be short lived!

Writes
------
- write() and commitWrite() to write to transaction or directly to disk
- both throw exceptions, most errors non-recoverable
- writes are synchronous and blocking

Refreshes
---------
- can refresh manually, with built-in runloop, or create your own
  runloop

Results
-------
- Results instance, with an interface similar to an Array

Chapter 11: JS React Native/Node/Electron SDK
=============================================
- supports React Native, Node, Typescript (a little), Electron, but NOT
  browser
- realm npm package

Data Models
-----------
- bool -> boolean
- int, float, double -> number
- string -> string
- data -> ArrayBuffer
- date -> Date
- pending: Decimal128, ObjectID
- primitives are not optional by default (no null/undefined)
- properties can store lists of the aforementioned types
- optional with "?" (not a typo, just a question mark character)
- create your schema from JS Objects with name/properties (and more)
  fields:
```
const ExampleSchema = {
    name: "Example",
    properties: {
       ex1: "string",
       ex2: "int?",
       ex3: "date[]"
    },
    primaryKey: "ex1"
}
```
- you can define properties with just the type, or as an object:
```
ex1: { type: "string", default: "cat", }
```
- instantiate your realm with a list of the objects in your schema:
```
let Realm = new Realm({schema: [ExampleSchema, OtherExample, ...]});
```
- can't change primary keys (pk) once added
- automatically index pk, but can index (int, string, bool, date)
- limit to index-able types also limits pk-able types
- { type: "linkingObjects", objectType: "otherObject", property: "ex1" }
  allows you to define a relation between two objects based on property
- make the relation property a list to create a one-to-many relationship

Connecting
----------
- to open synced realm, get valid logged in user, set config, open
- use sync for initializing a new Realm
- use async for existing realms
- sync property attributes to "openImmediately" or "downloadBeforeOpen",
  can customize behavior with newRealmFileBehavior and
  existingRealmFileBehavior

Queries
-------
- addlistener on Realm, RealmObject, Results, List to register
  notification callbacks
- notifications tell you indices of insertions, deletes, updates since
  last notification
- notifications are async

Writes
------
- wrap in Realm.write() transacation, wrap in try/catch for error
- write nested objects/relationships with a nested object in the written
  object
- push() to insert into lists
- update with primary key, update:true
- query -> set property -> write in one transaction to update
- can bulk update w/ loop in write transaction
- updates have (true OR "all") for create, "modified" for intelligent
  update, false for update-all-fields regardless of change
- delete() with primary key
- realm live lists can pop(), slice(), shift()
- deleteAll() can be used to reset the realm (CAUTION)

Refreshes
---------
- write transaction start advances version of read transactions
  - because of this, notifications can sometimes fire when your writes
    *begin* -- somewhat counterintuitive
- write transaction end & manual refresh also advance version

Results
-------
- queries return a Results object
  - live, auto-updates
  - if you use `for...in` or `for...of`, they will ALWAYS iterate over the
    list of objects matching the query when the iteration STARTED,
    even if items are modified/added/deleted during the iteration
- lists are not mutable (stay same length) but objects in them can be
  changed
- isValid exists for any result object, can tell you if it's changed
  under you or not
- query language "inspired by" NSPredicate (Apple/iOS)
  example:
```
  realm.objects("Dog")
       .filtered("color='tan' AND name BEGINSWITH 'B'")
```
- can use `LIKE, CONTAINS, BEGINSWITH, ENDSWITH` with `[c]` for case
  insensitive
- aggregates with @count, @size, @min, @max, @sum, @avg
- sort by multiple properties, including those of linked objects
  - sorted() for default sort, all sorts are extended latin only
- limit
- slice w/indexes for subset

Limitations
-----------
- React Native currently suffers from a bug wherein writes take too
  long, miss UI refresh, and force a manual UI refresh

Chapter 12: MongoDB Stitch
==========================

Architecture
------------
- serverless apps using Atlas as a data store
- Functions, triggers, auth all baked-in
- Stitch SDKs for mobile, browser platforms
- Stitch QueryAnywhere
- stitch app server: go monolith. All components are goroutines
- global apps config lives in a geo-sharded mongo server
- limitations on functions/graphql

SDKs
----
- iOS, Android, Node/Browser

GraphQL
-------
- automatically create GraphQL types/resolvers based on schema of Atlas
  data
- need to configure a role w/ access to GraphQL

Authentication
--------------
- rules

Chapter 13: MongoDB Realm
=========================

Sell
----
- bi-directional server-client sync
- simple, idiomatic SDKs
- functions, triggers, auth, data access controls
- serverless
- consumption-based payment model

Client Goals
------------
- address longstanding Realm issues
- align mongoDB/Realm
- Broaden: far future
    - new platforms: IoT, Flutter, Browser, Kotlin, Java Server
    - text/geo query support (and GeoJson)
    - new types: dicts, sets, maps, embedded objects
    - cascading deletes
    - inheritance
    - query nested arrays

Server Goals
------------
- development mode: auto schema
- ROS merging into stitch core server
- deprecating realm data connectors for MsSQL, Postgres
- auth/access all through Stitch
- Stitch SDKs merged into Realm SDK

MR Adapter
----------
- define Realm schema for MDB data: adapter automatically tansforms
  types not in Realm
- define which collections to sync
- change stream pushes events to ROS, which syncs to clients
- Realm Sync Adapter pushes all ROS events to Atlas
  - Sync Adapter knows consumer, so it doesn't filter Atlas' events back
    to Atlas
- Adapter is a separate process running alongside ROS and Atlas
- partitioning: "shard" collection into different realms
  - partition key: different values == different realms
  - only one partition key per stitch app: partition key can apply to
    multiple collections (you can sync each object type in one Realm to
    different collections in Atlas)

MR Translater ("Mr. T")
-----------------------
- module of MongoDB Stitch, written in Go
- operational transform is only method of conflict resolution
- no separate data copy
- single process, leverages Realm Sync where possible (acts like a
  client)
- MongoDB 4.4 only (added some small features to mongodb to support
  history<-> oplog sync)

Schema
------
- schema generation based on existing data (using roughly Stitch
  GraphQL schema generator)
- schema validation: inconsistent documents *can* exist in Atlas, but
  won't show up in Realm
- enable sync & all documents are scanned & valid documents added to
  sync state
- incompatible docments are ignored & logged, changes that occur during
  initialization are synced during a "catchup" phase after
  initialization
- generate schema code snippets for each SDK from Stitch
- if a document "falls out" of schema (in Atlas), it'll stop syncing.
  Fix it and it will resync.
- no dots in field names (but you wouldn't do that anyway, right?)

Authentication
--------------
- rules, users, system user, etc. all handled with current Stitch logic
- Realm -> RealmApp (just the mobile portion, whole thing is "Realm")
- no more Cloud URL, instead just a Stitch app ID

Chapter 14: Conclusion
======================

Things Realm Does Well
----------------------
- passive background sync between many clients and server
- auto conflict resolution when many clients interact with same data
- private/public/shared realms are easy to use, define, keep data
  silo'd and shared appropriately
- server-side integrations with Node/.NET/GraphQL based on notifications
- you can push data from clients or from the server and not worry about
  overwrites
- clients can work offline without worry/loss of functionality
- devs don't have to deal with the network when they write app logic

Things Realm Does Not Do So Well
--------------------------------
- Query Based Sync (RIP)
- can't store blobs/photos/audio easily: shouldn't insert things above
  5MB at all
- ephemeral connections ("hey, give me this blog post. Comment 'cool'.
  Never look at it again'") are tricky because Realm is based upon
  syncing entire realms at a time.
  - Stitch compliments this nicely with Functions
  - "offline first" doesn't work well with this anyway

FAQ:
----
- You tell me
