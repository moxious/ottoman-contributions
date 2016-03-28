== What we're using ottoman for

An ODM for couchbase. :)  But seriously, we have a lot of data we're pulling from an old relational database via an API, and then putting into JSON documents.  As such, we do end up with various RDBMS integer keys, not just things like UUIDs.

== Things we're thinking about/doing

(Listed in decreasing order of focus, most important stuff first)

=== Custom ID fields for "natural keys"

(Erik?)

=== Upsert

(Erik?)

Couchbase has upsert.  We want it in Ottoman.

=== Failure to build many n1ql indexes at once

Thread reported here, so as to not be repeated:
https://forums.couchbase.com/t/ottoman-n1ql-index-building-issue/7532

Short version, it looks like for building GSI / n1ql indexes, Ottoman needs to
move to a model of creating them all with {"defer_build":true} and then just
issue a BUILD INDEX command.

On top of that, there are various handlers in ottoman that are silently
swallowing errors.  Some errors (like index builds) come through in the "meta"
information, not in the "err" on the callback.   I.e. the callback for an
n1ql query has the signature (err, rows, meta).   Some ottoman callbacks
are only checking for err != null; when an index build fails, err === null,
but "meta" is telling you why.   (Arguably this is a problem with the Couchbase
node SDK, that index fail should be an actual error)

=== Ottoman Array Field Types have a bug

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

=== Promises

We use promises heavily, and we'd love to see a promisified Ottoman rather than
using callbacks.  In some cases we use `Promise.promisify`, but it'd be nice
to explore options for making a native promises option.

== Deferred Indexes

Not entirely sure what causes the "one-by-one" index creation but with n1ql
indexes, but I can replicate it.  Interested in exploring deferred index
building.

=== Query-defined methods on model instances

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

=== Couchbase, Ottoman, and Sync Gateway

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

=== Ottoman Test Suite

At present it looks under-specified; it tests mainly the Ottoman API but
doesn't do any integration testing with an actual couchbase server or with the
node SDK.  As a result, we're running into some of these troubles above which
are more integration issues.