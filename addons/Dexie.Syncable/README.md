# Dexie.Syncable.js

Enables two-way synchronization with remote database.

*NOTE! This package has been unmaintained for a looong time and might be retired. Some people go under the impression that Dexie Cloud would be based on this package, but it's not - it has its own sync system based on middlewares - all source for it is in addons/dexie-cloud.*

### Install
```
npm install dexie --save
npm install dexie-observable --save
npm install dexie-syncable --save
```

### Use
```js
import Dexie from 'dexie';
import 'dexie-syncable'; // will import dexie-observable as well.

// Use Dexie as normally - but you can also register your sync protocols though
// Dexie.Syncable.registerSyncProtocol() api as well as using the db.syncable api
// as documented here.

```

### Dependency Tree

 * **Dexie.Syncable.js**
   * [Dexie.Observable.js](https://dexie.org/docs/Observable/Dexie.Observable.js)
     * [Dexie.js](https://dexie.org/docs/Dexie/Dexie.js)
       * [IndexedDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API)
   * _An implementation of [ISyncProtocol](https://dexie.org/docs/Syncable/Dexie.Syncable.ISyncProtocol)_

### Tutorial

#### 1. Include Required Sources
In your HTML, make sure to include Dexie.js, Dexie.Observable.js, Dexie.Syncable.js and an implementation of [ISyncProtocol](https://dexie.org/docs/Syncable/Dexie.Syncable.ISyncProtocol).

    <html><head>
        <script src="dexie.min.js"></script>
        <script src="dexie-observable.min.js"></script>
        <script src="dexie-syncable.min.js"></script>
        <script src="WebSocketSyncProtocol.js"></script> <!-- Example implementation of ISyncProtocol -->
        ...
    </head><body>
    </body></html>
    
##### Usage with existing DB

In case you want to use Dexie.Syncable with your existing database, but do not want to use UUID based Primary Keys as described below, you will have to do a schema upgrade. Without it Dexie.Syncable will not be able to properly work.

```javascript
import Dexie from 'dexie';
import 'dexie-observable';
import 'dexie-syncable';
import 'your-sync-protocol-implementation';

var db = new Dexie('myExistingDb');
db.version(1).stores(... existing schema ...);

// Now, add another version, just to trigger an upgrade for Dexie.Syncable
db.version(2).stores({}); // No need to add / remove tables. This is just to allow the addon to install its tables.
```

#### 2. Use UUID based Primary Keys ($$)
Two way replication cannot use auto-incremented keys if any sync node should be able to create objects no matter if it is offline or online. Dexie.Syncable comes with a new syntax when defining your store schemas: the double-dollar prefix ($$). Similarly to the ++ prefix in Dexie (meaning auto-incremented primary key), the double-dollar prefix means that the key will be given a universally unique identifier (UUID), in string format (For example "9cc6768c-358b-4d21-ac4d-58cc0fddd2d6").

    var db = new Dexie("MySyncedDB");
    db.version(1).stores({
        friends: "$$oid,name,shoeSize",
        pets: "$$oid,name,kind"
    });

#### 3. Connect to Server
You must specify the URL of the server you want to keep in-sync with. This has to be done once in the entire database life-time, but doing it on every startup is ok as well, since it won't affect already connected URLs.

    // This example uses the WebSocketSyncProtocol included in earlier steps.
    db.syncable.connect ("websocket", "https://syncserver.com/sync");
    db.syncable.on('statusChanged', function (newStatus, url) {
        console.log ("Sync Status changed: " + Dexie.Syncable.StatusTexts[newStatus]);
    });

#### 4. Use Your Database
Query and modify your database as if it was a simple Dexie instance. Any changes will be replicated to the server and changes on the server or an other window will replicate back to you.

    db.transaction('rw', db.friends, function (friends) {
        friends.add({name: "Arne", shoeSize: 47});
        friends.where('shoeSize').above(40).each(function (friend) {
            console.log("Friend with shoeSize over 40: " + friend.name);
        });
    });

_NOTE: Transactions only provide the Atomicity part of the [ACID](http://en.wikipedia.org/wiki/ACID) properties when using 2-way synchronization. This is due to the fact that the syncronization phase may result in another change overwriting the changes. However, it's still meaningful to use the transaction() method for atomicity. Atomicity is guaranteed not only locally but also when synced to the server, meaning that a part of the changes will never commit on the server until all changes from the transaction have been synced. In practice, you cannot increment a counter in the database (for example) and expect it to be consistent, but you can have a guaranteed that if you add a sequence of objects, all or none of them will replicate._

### API Reference

#### Static Members

[Dexie.Syncable.registerSyncProtocol (name, protocolInstance)](https://dexie.org/docs/Syncable/Dexie.Syncable.registerSyncProtocol())
Define how to replicate changes with your type of server.

[Dexie.Syncable.Statuses](https://dexie.org/docs/Syncable/Dexie.Syncable.Statuses)
Enum of possible sync statuses, such as OFFLINE, CONNECTING, ONLINE and ERROR.

[Dexie.Syncable.StatusTexts](https://dexie.org/docs/Syncable/Dexie.Syncable.StatusTexts)
Text lookup for status numbers

#### Non-Static Methods and Events

[db.syncable.connect (protocol, url, options)](https://dexie.org/docs/Syncable/db.syncable.connect())
Create a persistend two-way sync connection with the given URL.

[db.syncable.disconnect (url)](https://dexie.org/docs/Syncable/db.syncable.disconnect())
Stop syncing with the given URL but keep revision states until next connect.

[db.syncable.delete(url)](https://dexie.org/docs/Syncable/db.syncable.delete())
Delete all states and change queue for given URL.

[db.syncable.list()](https://dexie.org/docs/Syncable/db.syncable.list())
List the URLs of each remote node we have a state saved for.

[db.syncable.on('statusChanged')](https://dexie.org/docs/Syncable/db.syncable.on('statusChanged'))
Event triggered when sync status changes.

[db.syncable.setFilter ([criteria], filter)](https://dexie.org/docs/Syncable/db.syncable.setFilter())
Ignore certain objects from being synced defined by the given filter.

[db.syncable.getStatus (url)](https://dexie.org/docs/Syncable/db.syncable.getStatus())
Get sync status for the given URL.

[db.syncable.getOptions (url)](https://dexie.org/docs/Syncable/db.syncable.getOptions())
Get the options object for the given URL.


### Source

[Dexie.Syncable.js](https://github.com/dexie/Dexie.js/blob/master/addons/Dexie.Syncable/src/Dexie.Syncable.js)

### Description

Dexie.Syncable enables synchronization with a remote database (of almost any kind). It has its own API [ISyncProtocol](https://dexie.org/docs/Syncable/Dexie.Syncable.ISyncProtocol).
The [ISyncProtocol](https://dexie.org/docs/Syncable/Dexie.Syncable.ISyncProtocol) is pretty straight-forward to implement.
The implementation of that API defines how client- and server- changes are transported between local and remote nodes. The API support both poll-patterns
(such as ajax calls) and direct reaction pattern (such as WebSocket or long-polling methods). See samples below for each pattern.

### Sample [ISyncProtocol](https://dexie.org/docs/Syncable/Dexie.Syncable.ISyncProtocol) Implementations
 * [https://github.com/nponiros/sync_client](https://github.com/nponiros/sync_client)
 * [AjaxSyncProtocol.js](https://github.com/dfahlander/Dexie.js/blob/master/samples/remote-sync/ajax/AjaxSyncProtocol.js)
 * [WebSocketSyncProtocol.js](https://github.com/dfahlander/Dexie.js/blob/master/samples/remote-sync/websocket/WebSocketSyncProtocol.js)

### Sample Sync Servers
 * [https://github.com/nponiros/sync_server](https://github.com/nponiros/sync_server)
 * [WebSocketSyncServer.js](https://github.com/dfahlander/Dexie.js/blob/master/samples/remote-sync/websocket/WebSocketSyncServer.js)
