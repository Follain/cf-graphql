# cf-graphql

[![travis build status](https://img.shields.io/travis/jelz/cf-graphql.svg)](https://travis-ci.org/jelz/cf-graphql)
[![npm version](https://img.shields.io/npm/v/cf-graphql.svg)](https://www.npmjs.com/package/cf-graphql)
[![npm downloads](https://img.shields.io/npm/dt/cf-graphql.svg)](https://www.npmjs.com/package/cf-graphql)
[![deps status](https://img.shields.io/david/jelz/cf-graphql.svg)](https://david-dm.org/jelz/cf-graphql)
[![dev deps status](https://img.shields.io/david/dev/jelz/cf-graphql.svg)](https://david-dm.org/jelz/cf-graphql?type=dev)
[![codecov coverage](https://img.shields.io/codecov/c/github/jelz/cf-graphql.svg)](https://codecov.io/gh/jelz/cf-graphql)

`cf-graphql` is a library that allows you to query your data stored in [Contentful](https://www.contentful.com/) with [GraphQL](http://graphql.org/). A schema and value resolvers are automatically generated out of an existing space.

Generated artifacts can be used with any node-based GraphQL server. The outcome of the project's main function call is an instance of the [`GraphQLSchema`](http://graphql.org/graphql-js/type/#graphqlschema) class.


## First steps

If you just want to see how it works, please follow the [Example](#example) section. If you want to get your hands dirty, install the package and follow the [Usage](#usage) section:

```
npm install --save cf-graphql
```


## Example

This repository contains a project example. To run it, clone the repository and execute:

```
nvm use # optional, but we prefer node v6.10
npm install
npm run example
```

Navigate to <http://localhost:4000> to access an IDE (GraphiQL). You can query my demo space there. Please refer the [Querying](#querying) section for more details.

To use your own Contentful space with the example, you have to provide:

- space ID
- CDA token
- CMA token

Please refer the ["Authentication" section](https://www.contentful.com/developers/docs/references/authentication/) of Contentful's documentation.

You can provide listed values with env variables:

```
SPACE_ID=some-space-id CDA_TOKEN=its-cda-token CMA_TOKEN=your-cma-token npm run example
```


## Usage

Let's assume we've required this module with `const cfGraphql = require('cf-graphql')`. To create a schema out of your space you need to call `cfGraphgl.createSchema(spaceGraph)`.

What is `spaceGraph`? It is a graph-like data structure containing descriptions of content types of your space which additionally provide some extra pieces of information allowing the library to create a GraphQL schema. To prepare this data structure you need to fetch raw content types data from the [CMA](https://www.contentful.com/developers/docs/references/content-management-api/) and then pass it to `cfGraphql.prepareSpaceGraph(rawCts)`:

```js
const client = cfGraphql.createClient({
  spaceId: 'some-space-id',
  cdaToken: 'its-cda-token',
  cmaToken: 'your-cma-token'
});

client.getContentTypes()
.then(cfGraphql.prepareSpaceGraph)
.then(spaceGraph => {
  // `spaceGraph` can be passed to `cfGraphql.createSchema`!
});
```

The last step is to use the schema with some server. A popular choice is [express-graphql](https://github.com/graphql/express-graphql). The only caveat is how the context is constructed. The library expects the `entryLoader` key of the context to be set to an instance created with `client.createEntryLoader()`:

```js
// skipped: `require` calls, Express app setup, `client` creation

// `spaceGraph` was fetched and prepared in the previous snippet:
const schema = cfGraphql.createSchema(spaceGraph);
// BTW, you shouldn't be doing it per request, once is fine

app.use('/graphql', graphqlHTTP(() => ({
  schema,
  context: {entryLoader: client.createEntryLoader()}
})));
```

[You can see a fully-fledged example here](./example/server.js).


## Querying

For each content type there are two root-level fields:

- a singular field accepts a required `id` argument and resolves to a single entity
- a collection field accepts an optional `q` argument and resolves to a list of entities; the `q` argument is a query string you could use with the [CDA](https://www.contentful.com/developers/docs/references/content-delivery-api/)

Assuming you've got two content types named `post` and `author` with listed fields, this query is valid:

```graphql
{
  authors {
    name
  }

  posts(q: "fields.rating[gt]=5") {
    title
    rating
  }

  post(id: "some-post-id") {
    title
    author
    comments
  }
}
```

Reference fields will be resolved to:

- a specific type, if there is a validation that allows only entries of some specific content type to be linked
- the `EntryType`, if there is no such constraint. The `EntryType` is an interface implemented by all the specific types

Example where the `author` field links only entries of one content type and the `related` field links entries of multiple content types:

```graphql
{
  posts {
    author {
      name
      website
    }

    related {
      ... on Tag {
        tagName
      }
      ... on Place {
        location
        name
      }
    }
  }
}
```

Backreferences (_backrefs_) are automatically created for links. Assume our `post` content type links to the `author` content type via a field named `author`. Getting an author of a post is easy, getting a list of posts by an author is not. `_backrefs` mitigate this problem:

```graphql
{
  authors {
    _backrefs {
      posts__via__author {
        title
      }
    }
  }
}
```

When using backreferences, there is a couple of things to keep in mind:

- backrefs may be slow; always test with a dataset which is comparable with what you've got in production
- backrefs are generated only when a reference field specifies a single allowed link content type
- `_backrefs` is prefixed with a single underscore
- `__via__` is surrounded with two underscores; you can read this query out loud like this: _"get posts that link to author via the author field"_


## Contributing

Issue reports and PRs are more than welcomed.


## License

MIT
