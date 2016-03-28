## What we're using ottoman for

An ODM for couchbase. :)  But seriously, we have a lot of data we're pulling from an old relational database via an API, and then putting into JSON documents.  As such, we do end up with various RDBMS integer keys, not just things like UUIDs.

## Topics

(Listed in decreasing order of focus, most important stuff first)

### Custom ID fields for "natural keys"

Refdoc indexes depend on data in their fields being unique, because the value is 
a key in couchbase and the value is the key for the document. Currently, the 
code will build an index out of data that is undefined, as in `Doc|undefined` 
and any other additions that include an undefined value will collide with this 
reference and error out. I have a very simple patch for this that simply skips 
undefined fields so that you can store as many items with the field as 
undefined as you would like. 

### Upsert

The newer code for a custom id field correctly allows the custom natural key to 
be the value for the key in couchbase itself. This means that as long as a 
refdoc is used for this field, the user can craft their own "upsert" function 
around ottoman to find the relevant document (if it exists) and save it in 
either case. Couchbase itself has an upsert function to accomplish this concept 
and we beleive it would be valuable for ottoman to expose this if an id is 
created without using the auto uuid generation.  

### Failure to build many n1ql indexes at once

Thread reported here, so as to not be repeated:
https://forums.couchbase.com/t/ottoman-n1ql-index-building-issue/7532

Short version, it looks like for building GSI / n1ql indexes, Ottoman needs to
move to a model of creating them all with ```{"defer_build":true}``` and then just
issue a `BUILD INDEX` command.

On top of that, there are various handlers in ottoman that are silently
swallowing errors.  Some errors (like index builds) come through in the "meta"
information, not in the "err" on the callback.   I.e. the callback for an
n1ql query has the signature (err, rows, meta).   Some ottoman callbacks
are only checking for err !# null; when an index build fails, err === null,
but "meta" is telling you why.   (Arguably this is a problem with the Couchbase
node SDK, that index fail should be an actual error)

Side note:  we observe errors in Ottoman when trying to do stuff with models
before ensureIndices() has called its callback.  Would like to talk about that
a bit, and it would be desirable that when that callback fires, we can be sure
the indexes are in place enough to be functional, since there is a lot of
follow-on code that happens, which will use them.

### Ottoman Array Field Types have a bug

If I specify a field like this:
`fooProperty: ['string']`

It should work, except it doesn't.  The error you get is
```
[15:20:19] Error: Inexpected field options type.
    at Ottoman._parseFieldType (/Users/mda/sw/ow-back/node_modules/ottoman/lib/ottoman.js:151:11)
```    

(Fun typo in the error message too)

Ottoman expects the inside of the array to be an object, like this:

`fooProperty: [ { type: 'string' } ]`

Except that the docs explain that the other way is OK.  So ottoman needs to
pick just one way of doing this, or support what's in the docs.

### Promises

We use promises heavily, and we'd love to see a promisified Ottoman rather than
using callbacks.  In some cases we use `Promise.promisify`, but it'd be nice
to explore options for making a native promises option.

We don't know how many other people are using Ottoman.  Making everything work
by promises would be a breaking change; a second option is a wrapper, i.e.
ottoman.Model vs. ottoman.PromiseModel where they have the same API, just one
works via Promises.  Open to discussion on other options.

### Query-defined methods on model instances

It'd be very nice to be able to define an n1ql query on a model instance, and
execute it as a JS function.  I.e. having a model Foo, I'd like to have an easy
way in the model definition to attach a function bar(), so that I can call
fooInstance.bar() and execute related n1ql queries.  As an example of a
'related n1ql query', imagine a document like this:

```
{ _type: 'Person', name: 'David', oldId: 25, parent: 20 }
```

I'd probably want to define my own getParent() that would execute a query that
would find another `Person` where `oldId: 20`.   Ottoman doesn't need to do
this for me, it just needs to make it possible for me to do this without
getting into prototype functions perhaps.

### Couchbase, Ottoman, and Sync Gateway

One of the problems I run into is that ottoman uses `_type` to distinguish
between the models. Because couchbase has to use the same logic as couchdb,
fields starting with are not allowed for sync. So this whole synchronization
stuff where couchbase (because of couchdb) is strong at, is not working
because of ottoman.  [Here is the discussion of this](https://forums.couchbase.com/t/sync-gateway-and-ottoman-driven-shadow-unable-to-sync/7344)

Current tentative fix is to make this work, we have a hand-hacked version of
ottoman that gets around many hard-coded `_type` instances; I asked our other
dev about this, he says the code is not in a state where a pull request would
be good, but removing dependence on `_type` or making it configurable would be
desirable.

### Bulk Insert

It would be nice to have a bulk create/save on Ottoman models.  When we do
data loading, we don't want to call save on each object, or post them
individually to a REST endpoint, because that's slow if you have thousands
of items.

Instead it would be nice to do a bulk save on hundreds or thousands at a time.

In the docs for server 4.0, there's a mention that this can be done in node.js,
but no code sample:

http://developer.couchbase.com/documentation/server/4.0/developer-guide/batching-operations.html

We know it's a possibility with .NET:
http://docs.couchbase.com/developer/dotnet-2.0/bulk-operations.html

And java:
http://docs.couchbase.com/developer/java-2.0/documents-bulk.html

### Ottoman Test Suite

At present it looks under-specified; it tests mainly the Ottoman API but
doesn't do any integration testing with an actual couchbase server or with the
node SDK.  As a result, we're running into some of these troubles above which
are more integration issues.
