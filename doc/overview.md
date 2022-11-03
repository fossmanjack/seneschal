### What does this need to do?

- Accept and distribute dispatches from clients to other clients
- Maintain a local realtime copy of all user data
- We need to do this because when a new user is created or given access to a datum, the "current" copy has to be sent before we can start dispatching to it
- Store dispatches on a per-client basis so that when a client dcs and reconnects, dispatches received during that time can be spooled out



### Program flow

- A client connects and authenticates against LDAP or via JWT
- (See The Problem [tm] for description)
- Seneschal sends "rehydrateBegin" packet to client on socket
- Seneschal receives all stored dispatches from connecting client
- Seneshcal adds stored dispatches to that client's queue
- Seneschal reduces queue by timestamp to a single dispatch for each oid
- Seneschal sends that dispatch out to all connected clients, and stores it for disconnected clients
- Seneschal sends "rehydrateEnd" packet to client on socket
- Seneschal flushes sending client queue
- Seneschal receives dispatch from client
- Seneschal updates its master copy and broadcasts dispatch to all other clients
- Broadcasts to disconnected clients are stored in database for future reconciliation

ClientID and UserID are different.  UserID is used to identify a user, who might be using multiple
devices, and ... honestly doesn't even need to be unique.  Though it probably will be, to allow for
username changes.  ClientID is attached to the various data objects so in order to keep things synced
between clients, has to be authoritative for the dispatches.

So, when a user logs in, the cuid and uuid are associated by Seneschal.  UUID is used to determine
access, group membership, login details, etc, while CUID is used as a persistent address for dispatches.

- A client connects with username:password
- Seneschal confirms login, associates CUID with UUID in user database, and returns JWT
- Socket ID is associated with CUID
- Seneschal receives dispatch, parses, updates "master" document/database, broadcasts dispatch to other CUIDs
- If a CUID hasn't been seen for a set timeout period (30 days?), delete its association and queue

On a client's first login, all its local data is sent to Seneschal for storage and further updates.  So how do we resolve it when a user logs in, logs out, logs back in?  This is Another Problem [tm].

A user can have access to a list even if they haven't yet logged in, and in any case anytime they're added to a list they'll need to receive the full list.  Right?  So.  That's not actually that hard -- dispatch the list object and the itemstore, history, and images for each associated item to the user.  This could take a small while if the list is hundreds of items long.

Ehem.  The user needs the entire itemstore, though.  Right?  Because maybe they want to add something to the list that already exists in the central repo but that they haven't seen locally.  So that's all well and good.  Everyone gets the entire item store but can't see all the lists.

Is there a way to change this?  To mark certain items as "private?"  I'm sure there is.  Everything is always dispatched to seneschal but it's only sent to another client if their UID or GID is on the access list, so that's just a check in the broadcast.  Lists are the same way.

- Seneschal receives dispatch for OUID
- Seneschal updates local data for OUID
- Seneschal checks access list for OUID and broadcasts to that list
- I'm sure we could leverage websockets "rooms" for this functionality
- But what about disconnected clients?  They all need the dispatches too
- Also, how to "delete" when access changes?  That's a dispatch too, right?  Just send a "delete" dispatch to all clients *not* on the list?  That should work.


### The Problem [tm]

Each user has a local copy of the central data so that they can work on that copy while disconnected from the network.  The problem comes with reconciling.  Let's say two users make changes to a datum while disconnected, or even when only one of them is disconnected.  Each updates their local copy and sends out the dispatch for the server to parse.  I guess we have to simply declare that the timestamp is authoritative?

- User 1, connected, dispatches a change to Item 1 at 0700
- The dispatch is stamped 0700 and stored on Seneschal
- User 2, disconnected, dispatches a change to Item 1 at 0720
- The dispatch is stamped 0720 and stored locally
- User 1, connected, dispatches another change at 0730
- The dispatch is stamped 0730 and stored on Seneschal

User 1 now has the 0730 version and User 2 has the 0720 version.  Seneschal has the 0700 and 0730 dispatches.

- User 2 connects at 0740 and sends 0720.
- Seneschal receives 0720, and now has 0700, 0720, and 0730 (current).
- I thought I was being clever, just updating the props that had been changed, but now I'm thinking it would actually be easier to simply send the entire reference item in the dispatch, every time.  That way the current is always current.  And it's not like it would be a major change to the app functionality.

See, if I make each dispatch simply replace the relevant object, then the latest dispatch for each object is the only one that has to be saved.

Alternatively ...

0700 changes name
0720 changes quantity and location
0730 changes location

We want the latest version of each prop to be the one displayed.  That's what we're getting at.  But how do we do this?

```
for prop in props; do
	if timestamp (update) > timestamp (reference)
		return (prop) = update (prop)
	else
		return (prop) = reference (prop)
done

dispatch return
```

This is server work, right?  The server has to figure out how to handle incoming dispatches that are older than now.

ob1 = { name: 'test', qty: '1', location: 'store' }

dispatch([ ob1, { qty: '2', location: 'aisle' } ]) <-- 0720

dispatch([ ob1, { location: 'food court' } ]) <-- 0730

Desired outcome: all clients show ob1 = { name: 'test', qty: '2', location: 'food court' }

First I guess reduce all the dispatches.  Right?  Play all the received dispatches back so that subsequent changes to the same object prop get overwritten by later changes.  Then, compare the reduced object to the reference object.  If the reduced object is older, just apply the reference object changes.  The problem comes when the reference object is newer but the reduced object has props that are individually newer than the reference object props.

So how to solve this?

Going back to above --

refOb -> {
	name: 'test', ---
	qty: '1', 0700
	location: 'food court', 0730
}

redOb -> {
	qty: '2', 0720
	location: 'aisle', 0720
}

I mean I guess the thing to do would be to shuffle the incoming dispatches into the server queue and then reduce by timestamp.  Right?  Then send the reduced dispatch with all the requisite changes to all clients?

So, reconciliation:

- seneschal receives stored dispatches from connecting client
- seneschal adds dispatches to client queue
- seneschal sorts client queue and reduces by uid
- seneschal broadcasts reduced dispatch to all connected clients, and to client queue to each disconnected client

Yeah I think this'll work.

User 2's client queue would have:

ob1: [ 0700, { qty: '1', location: 'store' }]
ob1: [ 0730, { location: 'food court' }]

and would then add:

ob1: [ 0720, { qty: '2', location: 'aisle' }]

Seneschal would sort the queue:

ob1: [ 0700, ... ]
ob1: [ 0720, ... ]
ob1: [ 0730, ... ]

Then reduce.  The result would be:

ob1: [ 0740, { qty: '2', location: 'food court' }]

Not the exact data structure, but it serves to illustrate.  Seneschal then dispatches
ob1: [ 0740, { qty: '2', location: 'food court' }] to all connected clients or puts it in
the client queue of all disconnected clients.

Note that we want to wait to reconcile for a particular client until login, so all
the dispatches have to be stored individually.  Otherwise it becomes impossible to
determine which is the "latest" data.

So the queue database has to be structured like:

uuid - oid - timestamp - dispatch object (JSON)

Actually ...

In order to avoid duplication, it's better to structure two databases:

Dispatch:
- id (unique)
- oid
- timestamp
- dispatch object (JSON)

Queue:
- uuid
- dispatch.id

So you can SELECT ALL from QUEUE where uuid = (userID), then use that to grab all the dispatches from
the DISPATCH DB by ID.


### Have to Learn

- Express talk to MySQL
- Express talk to LDAP
- JWTs with React Native
- Websocket "rooms"
- Doing everything over TLS

Everything can really be stored in MySQL, right?  Honestly we don't even really
need LDAP, but since I'm intending this to be used with Yuno it's best to keep
that as the default.
