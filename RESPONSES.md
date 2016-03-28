```
[3/28/16, 12:24:49 PM] Brett Lawson: -- Custom ID Fields --

Custom ID fields not working is more of a bug.  If your custom ID field is undefined though, that should not cause the whole key to be undefined.

Not actually sure what the bug is here.  undefined should only end up in the key if you have an undefined id field value, which shouldn't be possible.

[3/28/16, 12:26:30 PM] Brett Lawson: -- Upsert --

Unfortunately, this would not actually work.  In the create case, everything would be fine, but in the update case, it could potentially cause data to be removed that was already there.

This would require a significant amount of work and would be quite restrictive in it's use simply due to the nature of how Ottoman behaves with its indices and such.

[3/28/16, 12:27:07 PM] Brett Lawson: -- N1QL Indexing --

This is partly an issue with Ottoman, and partially an issue with N1QL's index builders.  Not sure why it won't let you queue up index builds automatically.

Simply enable deferred build on each index and then add a new query to the end of the execution list for 'BUILD INDEX'

[3/28/16, 12:27:34 PM] Brett Lawson: -- Array Field Types --

I have a vague idea why this might be failing.  Definitely a bug though.

Looks like this is just an issue with the translation from 'simplified' schema definitions to the full-fledged one we use internally.

[3/28/16, 12:28:10 PM] Brett Lawson: -- Promises --

I like promises, but I don't want to break everyones code.  It might be possible though to actually support both by having the promises returned from methods, and auto-binding a callback to that promise in the case that one is provided.

It would be worth it to take a look at the entire Ottoman API and identify if my suggestion of supporting promises as well as callbacks would be reasonable.

[3/28/16, 12:29:04 PM] Brett Lawson: -- Custom Query Methods --

The specific relationship you have defined here is automatically configurable actually.  If you want custom prototypes though, you can attach them directly to the models you've generated (with prototypes), just like you might do with any normal Javascript 'class'.

Two methods:

queries: {
  myParents: {
    by: 'numeric_id',
    using: 'parent_id'
  }
}

Second method:

someModel.prototype.fooMethod = function () { 
   // Do stuff
};

This method will show up on all model instances

More documentation around query/index definitions in models is needed, otherwise no work is really needed here.

[3/28/16, 12:31:01 PM] Brett Lawson: -- Sync Gateway --

This is a change that is more 'fundamental'.  There are some weird side-effects to making this configurable.  I am however in several meetings related to Couchbase Server / Couchbase Mobile interoperability, so we might find other ways around this.

Defer this one.

[3/28/16, 12:31:48 PM] Brett Lawson: -- Bulk Insert --

Bulk insert is not actually possible at the protocol level.  We can pipeline multiple requests at once (and we do).  The best you can do in regards to this is to provide 1 REST endpoint that accepts many things to save, and then it would call save many times in parallel.

No work necessary here.

[3/28/16, 12:32:41 PM] Brett Lawson: -- Ottoman Test Suite --

Definitely needs improvement.  Running Couchbase Server on Travis is non-trivial, so we do not currently run integration tests with Couchbase Server for our normal testing.

Improve as needed
```

