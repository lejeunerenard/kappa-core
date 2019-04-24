# kappa-core

> a small core for append-only log based programs

A lot like [flumedb][flumedb], but using
[multifeed](https://github.com/noffle/multifeed) as an append-only log base,
which is actually a *set* of append-only logs.

Pronounced *"capricorn"*.

## Status

*Experimental*, but functional.

## Usage

```js
var kappa = require('kappa-core')
var memdb = require('memdb')

var core = kappa('./log', { valueEncoding: 'json' })
var idx = memdb()

var sum = 0

var sumview = {
  api: {
    get: function (core, cb) {
      this.ready(function () {
        cb(null, sum)
      })
    }
  },
  map: function (msgs, next) {
    msgs.forEach(function (msg) {
      if (typeof msg.value === 'number') sum += msg.value
    })
    next()
  },

  // where to store and fetch the indexer's state (which log entries have been
  // processed so far)
  storeState: function (state, cb) { idx.put('state', state, cb) },
  fetchState: function (cb) { idx.get('state', cb) }
}

// the api will be mounted at core.api.sum
core.use('sum', 1, sumview)  // name the view 'sum' and consider the 'sumview' logic as version 1

core.feed('default', function (err, feed) {
  feed.append(1, function (err) {
    core.api.sum.get(function (err, value) {
      console.log(value) // 1
    })
  })
})
```

## API

```js
var kappa = require('kappa-core')
```

### var core = kappa(storage, opts)

Create a new kappa-core database.

- `storage` is an instance of
  [random-access-storage](https://github.com/random-access-storage). If a string
  is given,
  [random-access-file](https://github.com/random-access-storage/random-access-storage)
  is used with the string as the filename.
- Valid `opts` include:
  - `valueEncoding`: a string describing how the data will be encoded.
  - `multifeed`: A preconfigured instance of [noffle/multifeed](https://github.com/noffle/multifeed)

### var feed = core.feed(name, cb)

Create or get a local writable feed called `name`. If it already existed, it is
returned. A feed is an instance of
[hypercore](https://github.com/mafintosh/hypercore).

### var feeds = core.feeds()

An array of all hypercores in the kappa-core. Check a feed's `key` to
find the one you want, or check its `writable` / `readable` properties.

Only populated once `core.ready(fn)` is fired.

### core.use(name[, version], view)

Install a view called `name` to the kappa-core instance. A view is an object of
the form

```js
{
  api: {
    someSyncFunction: function (core) { return ... },
    someAsyncFunction: function (core, cb) { process.nextTick(cb, ...) }
  },

  map: function (msgs, next) {
    msgs.forEach(function (msg) {
      // ...
    })
    next()
  },

  fetchState: function (cb) { ... },
  storeState: function (state, cb) { ... }
}
```

The kappa-core instance `core` is always is bound to `this` in all of the `api`
functions you define.

`version` is an integer that represents what version you want to consider the
view logic as. Whenever you change it (generally by incrementing it by 1), the
underlying data generated by the view will be wiped, and the view will be
regenerated again from scratch. This provides a means to change the logic or
data structures of a view over time in a way that is future-compatible.

The `{fetch,store}State` functions are optional: they tell the view where to
store its state information about what log entries have been indexed thus far.
If not passed in, they will be stored in memory (i.e. reprocessed on each fresh
run of the program). You can use any backend you want (like leveldb) to store
the `Buffer` object `state`.

There are also the following optional `opts`:

- `inedxed`: a function to run whenever a new batch of messages have been
  indexed & written to storage. Receives an array of messages.

### core.ready(viewNames, cb)

Wait until all views named by `viewNames` are caught up. e.g.

```
// one
core.ready('sum', function () { ... })

// or several
core.ready(['kv', 'refs', 'spatial'], function () { ... })
```

If viewNames is `[]` or not included, all views will be waited on.

### core.pause([viewNames], [cb])

Pause some or all of the views' indexing process. If no `viewNames` are given,
they will all be paused. `cb` is called once the views finish up any entries
they're in the middle of processing and are fully stopped.

### core.resume([viewNames])

Resume some or all paused views. If no `viewNames` is given, all views are
resumed.

### core.replicate([opts])

Create a duplex replication stream. `opts` are passed in to
[multifeed](https://github.com/noffle/multifeed)'s API of the same name.

### core.on('error', function (err) {})

Event emitted when an error within kappa-core has occurred. This is very
important to listen on, lest things suddenly seem to break and it's not
immediately clear why.

## Install

With [npm](https://npmjs.org/) installed, run

```
$ npm install kappa-core
```

## Useful view modules

Here are some useful modules that play well with kappa-core for building views:

- [unordered-materialized-bkd](https://github.com/digidem/unordered-materialized-bkd): spatial index
- [unordered-materialized-kv](https://github.com/digidem/unordered-materialized-kv): key/value store
- [unordered-materialized-backrefs](https://github.com/digidem/unordered-materialized-backrefs): back-references

## Why?

[flumedb][flumedb] presents an ideal small core API for an append-only log:
append new data, and build (versioned) views over it. kappa-core copies this
gleefully, but with two major differences:

1. [hypercore][hypercore] is used for feed (append-only log) storage
2. views are built in out-of-order sequence

hypercore provides some very useful superpowers:

1. all data is cryptographically associated with a writer's public key
2. partial replication: parts of feeds can be selectively sync'd between peers,
instead of all-or-nothing, without loss of cryptographic integrity

Building views in arbitrary sequence is more challenging than when order is
known to be topographic, but confers some benefits:

1. most programs are only interested in the latest values of data; the long tail
of history can be traversed asynchronously at leisure after the tips of the
feeds are processed
2. the views are tolerant of partially available data. Many of the modules
listed in the section below depend on *topographic completeness*: all entries
referenced by an entry **must** be present for indexes to function. This makes
things like the equivalent to a *shallow clone* (think [git][git-shallow]),
where a small subset of the full dataset can be used and built on without
breaking anything.

## Acknowledgments

kappa-core is built atop ideas from a huge body of others' work:

- [flumedb][flumedb]
- [secure scuttlebutt](http://scuttlebutt.nz)
- [hypercore][hypercore]
- [hyperdb](https://github.com/mafintosh/hyperdb)
- [forkdb](https://github.com/substack/forkdb)
- [hyperlog](https://github.com/mafintosh/hyperlog)
- a harmonious meshing of ideas with @substack in spain

## Further Reading

- [kappa architecture](http://kappa-architecture.com)

## License

ISC

[hypercore]: https://github.com/mafintosh/hypercore
[flumedb]: https://github.com/flumedb/flumedb
[git-shallow]: https://www.git-scm.com/docs/gitconsole.log(one#gitconsole.log(one---depthltdepthgt)
