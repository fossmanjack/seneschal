### Overall Structure

- **Seneschal** -- the back-end Express/Mongo combo, handles data management
- **Manciple** -- shopping list application
- **Pantrist** -- household inventory management
- **HeadChef** -- recipe book

Each is a React/React Native app and shares user data

I'm actually thinking about building this as a NextCloud-style suite of applications.
They can all tie into the Seneschal server and use the same user store, and you can
install the plugins you want.

Like, Seneschal (or maybe Castellan or Steward) ... no, Seneschal will be the frame
app that provides the user management interface and each of the addons will plug into
Seneschal.  We'll call the back end Quartermaster or something.

### App Design

The apps should all be able to use the same general front-end schema, so that'll make coding
them all easier.

It seems like I should be able to run a React app within a React app, right?  So
if that's the case the web stuff can be loaded on the fly.  But what about the RN
apps?  Can secure storage be shared among apps, to use the same credentials?  Can
I code the various plugins to poll the main app for credentials?

### Seneschal

Node Express interface for MongoDB.  Maintains:
- Users: users and permissions
- ItemStore: repository for item information
- Pantry: items in the household
- Lists: lists of items to purchase and users with access
- History: purchase history
- Chirurgeon: medical info
- Steward: chores and household tasks

Seneschal receives dispatches and holds them in a journal.

{
	timestamp,
	slice,
	action,
	actionID,
	payload,
	user,
	client
}

Each time a client connects, it sends a lastConnect and asks if there's anything newer.
Seneschal grabs what's newer, checks what the user has access to, and sends it all
back as JSON-stringified array of objects sorted by timestamp, oldest to newest.
The client then dispatches all that, then processes it all, then sends its newest
dispatch.  This would be slow as fuck and will never work.

### Manciple

Interacts with Lists, Pantry, ItemStore, History collections

Something to consider: create a separate scanner app that reads bar and QR codes,
checks against the current grocery list, and (adds and) checks off the item if it
can find the UPC or prompts the user to add or select the item from the list if
it can't find it, auto-populates the upc field, and updates history accordingly.
Potentially just build it into Manciple as "check-in mode"

### Pantrist

Interacts with Pantry

### HeadChef

Interacts with Pantry, ItemStore

We'll need to store quantities as <number>, <string> -- because I'm not about to
write the logic to convert "1 handful" and "1 dash" and combine them into a total
for a grocery list.

### <Chores manager>

Probably maintains a separate collection, basically a more-advanced to-do list
with user-assignable jobs

Can have a "verify" function for parents and kids -- the kids check off they did
a chore and the parent can go check on it.  Checking with your account stamps it
with your name and the time.

"Chores" (classes) and "Tasks" (objects) -> each "Task" is an instantiated, time-
stamped Chore with a due date.  A Task is added to a Chore List and checked off
when done by whoever does it, with timestamp.  Permissions are granular -- you
can have "auto-verify" and "can verify" enabled separately.  Like if you trust one kid,
give them "auto-verify" but not "can verify," and if you've got a parent who also
needs checking up on you can enable "can verify" but not "auto-verify."

### Chirurgeon

Medical records and period tracker -- height, weight, blood pressure, mood, medication
reminder, period, whatever.  Stores data encrypted -- figure out how to share across clients?
Maybe using JWTs?  Also have a kill switch that deletes the encryption keys before deleting
the data.

### Calendar/scheduling

### Alexa integration

The holy grail would be telling Alexa to add something to a shopping list or
allocate a chore or whatever and making it happen

### Authenticate against LDAP!

This would allow YunoHost integration

### Dockerify seneschal
