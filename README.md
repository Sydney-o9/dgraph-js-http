# dgraph-js-http [![npm version](https://img.shields.io/npm/v/dgraph-js-http.svg?style=flat)](https://www.npmjs.com/package/dgraph-js-http) [![Build Status](https://img.shields.io/travis/dgraph-io/dgraph-js-http/master.svg?style=flat)](https://travis-ci.org/dgraph-io/dgraph-js-http) [![Coverage Status](https://img.shields.io/coveralls/github/dgraph-io/dgraph-js-http/master.svg?style=flat)](https://coveralls.io/github/dgraph-io/dgraph-js-http?branch=master)

A Dgraph client implementation for javascript using HTTP. It supports both
browser and Node.js environments.

This client follows the [Dgraph Javascript GRPC client][grpcclient] closely.

[grpcclient]: https://github.com/dgraph-io/dgraph-js

Before using this client, we highly recommend that you go through [docs.dgraph.io],
and understand how to run and work with Dgraph.

[docs.dgraph.io]:https://docs.dgraph.io

## Table of contents

- [Install](#install)
- [Quickstart](#quickstart)
- [Using a client](#using-a-client)
  - [Create a client](#create-a-client)
  - [Alter the database](#alter-the-database)
  - [Create a transaction](#create-a-transaction)
  - [Run a mutation](#run-a-mutation)
  - [Run a query](#run-a-query)
  - [Commit a transaction](#commit-a-transaction)
  - [Debug mode](#debug-mode)
- [Development](#development)
  - [Building the source](#building-the-source)
  - [Running tests](#running-tests)

## Install

Install using npm:

```sh
npm install dgraph-js-http --save
```

or yarn:

```sh
yarn add dgraph-js-http
```

You will also need a Promise polyfill for
[older browsers](http://caniuse.com/#feat=promises) and Node.js v5 and below.
We recommend [taylorhakes/promise-polyfill](https://github.com/taylorhakes/promise-polyfill)
for its small size and Promises/A+ compatibility.

## Quickstart

Build and run the [simple] project in the `examples` folder, which
contains an end-to-end example of using the Dgraph javascript HTTP client. Follow
the instructions in the README of that project.

[simple]: https://github.com/dgraph-io/dgraph-js-http/tree/master/examples/simple

## Using a client

### Create a client

A `DgraphClient` object can be initialised by passing it a list of
`DgraphClientStub` clients as variadic arguments. Connecting to multiple Dgraph
servers in the same cluster allows for better distribution of workload.

The following code snippet shows just one connection.

```js
const dgraph = require("dgraph-js-http");

const clientStub = new dgraph.DgraphClientStub(
  // addr: optional, default: "http://localhost:8080"
  "http://localhost:8080",
);
const dgraphClient = new dgraph.DgraphClient(clientStub);
```

To facilitate debugging, [debug mode](#debug-mode) can be enabled for a client.

### Alter the database

To set the schema, pass the schema to `DgraphClient#alter(Operation)` method.

```js
const schema = "name: string @index(exact) .";
await dgraphClient.alter({ schema: schema });
```

> NOTE: Many of the examples here use the `await` keyword which requires
> `async/await` support which is not available in all javascript environments.
> For unsupported environments, the expressions following `await` can be used
> just like normal `Promise` instances.

`Operation` contains other fields as well, including drop predicate and drop all.
Drop all is useful if you wish to discard all the data, and start from a clean
slate, without bringing the instance down.

```js
// Drop all data including schema from the Dgraph instance. This is useful
// for small examples such as this, since it puts Dgraph into a clean
// state.
await dgraphClient.alter({ dropAll: true });
```

### Create a transaction

To create a transaction, call `DgraphClient#newTxn()` method, which returns a
new `Txn` object. This operation incurs no network overhead.

It is good practise to call `Txn#discard()` in a `finally` block after running
the transaction. Calling `Txn#discard()` after `Txn#commit()` is a no-op
and you can call `Txn#discard()` multiple times with no additional side-effects.

```js
const txn = dgraphClient.newTxn();
try {
  // Do something here
  // ...
} finally {
  await txn.discard();
  // ...
}
```

### Run a mutation

`Txn#mutate(Mutation)` runs a mutation. It takes in a `Mutation` object, which
provides two main ways to set data: JSON and RDF N-Quad. You can choose whichever
way is convenient.

We define a person object to represent a person and use it in a `Mutation` object.

```js
// Create data.
const p = {
    name: "Alice",
};

// Run mutation.
await txn.mutate({ setJson: p });
```

For a more complete example with multiple fields and relationships, look at the
[simple] project in the `examples` folder.

For setting values using N-Quads, use the `setNquads` field. For delete mutations,
use the `deleteJson` and `deleteNquads` fields for deletion using JSON and N-Quads
respectively.

Sometimes, you only want to commit a mutation, without querying anything further.
In such cases, you can use `Mutation#commitNow = true` to indicate that the
mutation must be immediately committed.

```js
// Run mutation.
await txn.mutate({ setJson: p, commitNow: true });
```

### Run a query

You can run a query by calling `Txn#query(string)`. You will need to pass in a
GraphQL+- query string. If you want to pass an additional map of any variables that
you might want to set in the query, call `Txn#queryWithVars(string, object)` with
the variables object as the second argument.

The response would contain the `data` field, `Response#data`, which returns the response
JSON.

Let’s run the following query with a variable $a:

```console
query all($a: string) {
  all(func: eq(name, $a))
  {
    name
  }
}
```

Run the query and print out the response:

```js
// Run query.
const query = `query all($a: string) {
  all(func: eq(name, $a))
  {
    name
  }
}`;
const vars = { $a: "Alice" };
const res = await dgraphClient.newTxn().queryWithVars(query, vars);
const ppl = res.data;

// Print results.
console.log(`Number of people named "Alice": ${ppl.all.length}`);
ppl.all.forEach((person) => console.log(person.name));
```

This should print:

```console
Number of people named "Alice": 1
Alice
```

### Commit a transaction

A transaction can be committed using the `Txn#commit()` method. If your transaction
consisted solely of calls to `Txn#query` or `Txn#queryWithVars`, and no calls to
`Txn#mutate`, then calling `Txn#commit()` is not necessary.

An error will be returned if other transactions running concurrently modify the same
data that was modified in this transaction. It is up to the user to retry
transactions when they fail.

```js
const txn = dgraphClient.newTxn();
try {
  // ...
  // Perform any number of queries and mutations
  // ...
  // and finally...
  await txn.commit();
} catch (e) {
  if (e === dgraph.ERR_ABORTED) {
    // Retry or handle exception.
  } else {
    throw e;
  }
} finally {
  // Clean up. Calling this after txn.commit() is a no-op
  // and hence safe.
  await txn.discard();
}
```

### Debug mode

Debug mode can be used to print helpful debug messages while performing alters,
queries and mutations. It can be set using the`DgraphClient#setDebugMode(boolean?)`
method.

```js
// Create a client.
const dgraphClient = new dgraph.DgraphClient(...);

// Enable debug mode.
dgraphClient.setDebugMode(true);
// OR simply dgraphClient.setDebugMode();

// Disable debug mode.
dgraphClient.setDebugMode(false);
```

## Development

### Building the source

```sh
npm run build
```

### Running tests

Make sure you have a Dgraph server running on localhost before you run this task.

```sh
npm test
```