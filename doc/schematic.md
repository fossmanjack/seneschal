### Endpoints

- /users --> verify against LDAP (and, later, localstore or others)
- /lists
- /items
- /itemHistory
- /images
- /collections

### MongoDB schema

collections: lists, items, images, history, access, users

### MySQL schema

tables: lists, [inventory, staples,] items, [history, images,] access, users
