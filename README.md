# Restivus 
#### REST APIs for the Best of Us!

Restivus makes building REST APIs in Meteor 0.9.0+ easier than ever before! The package is inspired
by [RestStop2][reststop2-docs] and [Collection API](https://github.com/crazytoad/meteor-collectionapi),
and is built on top of [Iron Router][iron-router]'s server-side routing to provide:
- A simple interface for creating REST APIs
- Easily setup CRUD endpoints for Mongo Collections
- User authentication via the API
  - Optional login and logout endpoints
  - Access to `this.user` in authenticated endpoints
  - Custom authentication if needed
- Role permissions for limiting access to specific endpoints
  - Works alongside the [`alanning:roles`][alanning-roles] package - Meteor's accepted role
    permission package
- And coming soon:
  - Basic versioning of routes and endpoints
  - JSON PATCH support on collections
  - Autogenerated OPTIONS endpoint on routes
  - Pre and post hooks on all endpoints

## Installation

You can install Restivus using Meteor's package manager:
```bash
> meteor add nimble:restivus
```

And to update Restivus to the latest version:
```bash
> meteor update nimble:restivus
```

## Quick Start

###### CoffeeScript:
```coffeescript
Items = new Mongo.Collection 'items'

if Meteor.isServer

  # Global API configuration
  Restivus.configure
    useAuth: true
    prettyJson: true

  # Generates: GET, POST, DELETE on /api/items and GET, PUT, DELETE on
  # /api/items/:id for Items collection
  Restivus.addCollection Items

  # Generates: GET, POST on /api/users and GET, DELETE /api/users/:id for
  # Meteor.users collection
  Restivus.addCollection Meteor.users,
    excludedEndpoints: ['deleteAll', 'put']
    routeOptions:
      authRequired: true
    endpoints:
      post:
        authRequired: false
      delete:
        roleRequired: 'admin'

  # Maps to: /api/posts/:id
  Restivus.addRoute 'posts/:id', authRequired: true,
    get: ->
      post = Posts.findOne @urlParams.id
      if post
        status: 'success', data: post
      else
        statusCode: 404
        body: status: 'fail', message: 'Post not found'
    post:
      roleRequired: ['author', 'admin']
      action: ->
        post = Posts.findOne @urlParams.id
        if post
          status: "success", data: post
        else
          statusCode: 400
          body: status: "fail", message: "Unable to add post"
    delete:
      roleRequired: 'admin'
      action: ->
        if Posts.remove @urlParams.id
          status: "success", data: message: "Item removed"
        else
          statusCode: 404
          body: status: "fail", message: "Item not found"
```

###### JavaScript:
```javascript
Items = new Mongo.Collection('items');

if (Meteor.isServer) {

  // Global API configuration
  Restivus.configure({
    useAuth: true,
    prettyJson: true
  });

  // Generates: GET, POST, DELETE on /api/items and GET, PUT, DELETE on
  // /api/items/:id for Items collection
  Restivus.addCollection(Items);

  // Generates: GET, POST on /api/users and GET, DELETE /api/users/:id for
  // Meteor.users collection
  Restivus.addCollection(Meteor.users, {
    excludedEndpoints: ['deleteAll', 'put'],
    routeOptions: {
      authRequired: true
    },
    endpoints: {
      post: {
        authRequired: false
      },
      delete: {
        roleRequired: 'admin'
      }
    }
  });

  // Maps to: /api/posts/:id
  Restivus.addRoute('posts/:id', {authRequired: true}, {
    get: function () {
      var post = Posts.findOne(this.urlParams.id);
      if (post) {
        return {status: 'success', data: post};
      }
      return {
        statusCode: 404,
        body: {status: 'fail', message: 'Post not found'}
      };
    },
    post: {
      roleRequired: ['author', 'admin'],
      action: function () {
        var post = Posts.findOne(this.urlParams.id);
        if (post) {
          return {status: "success", data: post};
        }
        return {
          statusCode: 400,
          body: {status: "fail", message: "Unable to add post"}
        };
      }
    },
    delete: {
      roleRequired: 'admin',
      action: function () {
        if (Posts.remove(this.urlParams.id)) {
          return {status: "success", data: {message: "Item removed"}};
        }
        return {
          statusCode: 404,
          body: {status: "fail", message: "Item not found"}
        };
      }
    }
  });
}
```

## Table of Contents

- [Upgrading from 0.5.x](#upgrading-from-05x)
- [Upgrading from 0.6.0](#upgrading-from-060)
- [Terminology](#terminology)
- [Writing a Restivus API](#writing-a-restivus-api)
  - [Configuration Options](#configuration-options)
  - [Defining Collection Routes](#defining-collection-routes)
    - [Collection](#collection)
    - [Collection Options](#collection-options)
      - [Route Configuration](#route-configuration)
      - [Endpoint Configuration](#endpoint-configuration)
    - [Request and Response Structure](#request-and-response-structure)
    - [Users Collection Endpoints](#users-collection-endpoints)
  - [Defining Custom Routes](#defining-custom-routes)
    - [Path Structure](#path-structure)
    - [Route Options](#route-options)
    - [Defining Endpoints](#defining-endpoints)
      - [Endpoint Configuration](#endpoint-configuration-1)
      - [Endpoint Context](#endpoint-context)
      - [Response Data](#response-data)
  - [Documenting Your API](#documenting-your-api)
- [Consuming a Restivus API](#consuming-a-restivus-api)
  - [Basic Usage](#basic-usage)
  - [Authenticating](#authenticating)
  - [Authenticated Calls](#authenticated-calls)
- [Resources](#resources)
  - [Change Log](#change-log)
  - [Contributing](#contributing)

## Upgrading from 0.5.x

Restivus v0.6.1 brings support for easily generating REST endpoints for your Mongo Collections, and
with that comes a few API-breaking changes:

- The `Restivus.add()` method has been changed to `Restivus.addRoute()` for clarity, now that the
  `Restivus.addCollection()` method has been added.
- All responses generated by Restivus now follow the [JSend] format, with one minor tweak: failures
  have an identical structure to errors (check out the [JSend documentation][jsend] for more details
  on the response structure, which will be demonstrated throughout these docs). Just to be clear,
  this only affects responses that are automatically generated by Restivus. Any responses you
  manually created will remain unaffected.

Please see the [change log][restivus-change-log] for more information about the changes between each
version.

## Upgrading from 0.6.0

**_WARNING!_ Do not use v0.6.0! Please upgrade to v0.6.1 or above.**

v0.6.0 does not allow you to work with existing collections, or access collections created in
Restivus elsewhere in your app, which is not the intended behavior. Since Meteor expects you to
construct each collection only once (using `new Mongo.Collection()`), and store that globally,
Restivus now requires that you pass an existing collection in `Restivus.addCollection()`. Please
check out the updated section on [defining collection routes](#defining-collection-routes) for a
detailed breakdown of the parameters to `Restivus.addCollection()`.

## Terminology

Just to clarify some terminology that will be used throughout these docs:

_**(HTTP) Method:**_
- The type of HTTP request (e.g., GET, PUT, POST, etc.)

_**Endpoint:**_
- The function executed when a request is made at a given URL path for a specific HTTP method

_**Route:**_
- A URL path and its set of configurable endpoints

# Writing A Restivus API

Restivus is a **server-only** package. Attempting to access any of its methods from the client will 
result in an error.

## Configuration Options

The following configuration options are available with `Restivus.configure` (must be called once,
but all properties are optional):

##### `apiPath`
- _String_
- Default: `'api/'`
- The base path for your API. If you use `'api'` and add a route called `'users'`, the URL will be
  `https://yoursite.com/api/users/`.

##### `defaultHeaders`
- _Object_
- Default: `{ 'Content-Type': 'application/json' }`
- The response headers that will be returned from every endpoint by default. These can be overridden 
  by [returning `headers` of the same name from any endpoint](#response-data). 

##### `useAuth`
- _Boolean_
- Default: `false`
- If `true`, `POST /login` and `GET /logout` endpoints are added to the API. You can access
  `this.user` and `this.userId` in [authenticated](#authenticating) endpoints.

##### `auth`
- _Object_
  - `token`  _String_
    - Default: `'services.resume.loginTokens.hashedToken'`
    - The path to the auth hashed token in the `Meteor.user` document. This location will be checked for a
      matching token if one is returned in `auth.user()`.
  - `user`  _Function_
    - Default: Get user ID and auth token from `X-User-Id` and `X-Auth-Token` headers
        ```javascript
        function() {
          return {
            userId: this.request.headers['x-user-id'],
            token: this.request.headers['x-auth-token']
          };
        }
        ```
        
    - Provides one of two levels of authentication, depending on the data returned. The context
      within this function is the [endpoint context](#endpoint-context) without `this.user` and
      `this.userId` (well, that's what we're working on here!). Once the user authentication
      completes successfully, the authenticated user and their ID will be attached to the [endpoint
      context](#endpoint-context). The two levels of custom authentication and their required return
      data are:
      - Partial auth
        - `userId`: The ID of the user being authenticated
        - `token`: The auth token to be verified
        - If both a `userId` and `token` are returned, Restivus will hash the token, then,
          authentication will succeed if the `hashed token`
          exists in the given `Meteor.user` document at the location specified in `auth.token`
      - Complete auth
        - `user`: The fully authenticated `Meteor.user`
        - This is your chance to completely override the user authentication process. If a `user` is
          returned, any `userId` and `token` will be ignored, as it's assumed that you have already
          successfully authenticated the user (by whatever means you deem necessary). The given user
          is simply attached to the [endpoint context](#endpoint-context), no questions asked.

##### `prettyJson`
- _Boolean_
- Default: `false`
- If `true`, render formatted JSON in response.

##### `onLoggedIn`
- _Function_
- Default: `undefined`
- A hook that runs once a user has been successfully logged into their account via the `/login`
  endpoint. [Context](#endpoint-context) is the same as within authenticated endpoints. Any
  returned data will be added to the response body as `data.extra` (coming soon).

##### `onLoggedOut`
- _Function_
- Default: `undefined`
- Same as onLoggedIn, but runs once a user has been successfully logged out of their account via
  the `/logout` endpoint. [Context](#endpoint-context) is the same as within authenticated
  endpoints. Any returned data will be added to the response body as `data.extra` (coming soon).

##### `useClientRouter`
- _Boolean_
- Default: `true`
- If `false`, disable Iron Router on the client. This is recommended if you're not using Iron Router
  for your client-side routing. **Note: Since this needs to be configured on the client, you must
  call `Restivus.configure()` in a file available on both the client and server (e.g., common.js)
  for this to actually take effect.**

```coffeescript
  Restivus.configure
    useAuth: true
    apiPath: 'my-api/'
    prettyJson: true
    auth:
      token: 'auth.apiKey'
      user: ->
        userId: @request.headers['user-id']
        token: @request.headers['login-token']
    onLoggedIn: -> console.log "#{@user.username} (#{@userId}) logged in"
    onLoggedOut: -> console.log "#{@user.username} (#{@userId}) logged out"
    useClientRouter: false
```

##### `enableCors`
- _Boolean_
- Default: `true`
- If true, enables cross-origin resource sharing ([CORS]). This allows your API to receive requests 
  from _any_ domain (when `false`, the API will only accept requests from the domain where the API 
  is being hosted. _Note: Only applies to requests originating from browsers)._

## Defining Collection Routes

One of the most common uses for a REST API is exposing a set of operations on your collections.
Well, you're in luck, because this is almost _too easy_ with Restivus! All available REST endpoints
(except `patch` and `options`, for now) can be generated for a Mongo Collection using
`Restivus.addCollection()`. This generates two routes by default:

**`/api/<collection>`**
- Operations on the entire collection
-  `GET`,`POST`, and `DELETE`

**`/api/<collection>/:id`**
- Operations on a single entity within the collection
- `GET`, `PUT`, and `DELETE`

### Collection

The first - and only required - parameter of `Restivus.addCollection()` is a Mongo Collection.
Please check out the [Meteor docs](http://docs.meteor.com/#/full/collections) for more on creating
collections. The `Meteor.users` collection will have [special endpoints]
(#users-collection-endpoints) generated.

### Collection Options

Route and endpoint configuration options are available in `Restivus.addCollection()` (as the 2nd,
optional parameter).

#### Route Configuration

The top level properties of the options apply to both routes that will be generated
(`/api/<collection>` and `/api/<collection>/:id`):

##### `path`
- _String_
- Default: Name of the collection (the name passed to `new Mongo.Collection()`, or `'users'` for
  `Meteor.users`)
- The base path for the generated routes. Given a path `'other-path'`, routes will be generated at
  `'api/other-path'` and `'api/other-path/:id'`

##### `routeOptions`
- _Object_
- `authRequired` _Boolean_
  - Default: `false`
  - If true, all endpoints on these routes will return a `401` if the user is not properly
    [authenticated](#authenticating).
- `roleRequired` _String or Array of Strings_
  - Default: `undefined` (no role required)
  - The acceptable user roles for all endpoints on this route (e.g., `'admin'`, `['admin', 'dev']`).
    Additional role permissions can be defined on specific endpoints. If the authenticated user does
    not belong to at least one of the accepted roles, a `401` is returned. Since a role cannot be
    verified without an authenticated user, setting the `roleRequired` implies `authRequired: true`,
    so that option can be omitted without any consequence. For more on setting up roles, check out
    the [`alanning:roles`][alanning-roles] package.

##### `excludedEndpoints`
- _String or Array of Strings_
- Default: `undefined`
- The names of the endpoints that should _not_ be generated (see the `endpoints` option below for a
  complete list of endpoint names).

##### `endpoints`
- _Object_
- Default: `undefined` (all available endpoints generated)
- Each property of this object corresponds to a REST endpoint. In addition to the
  `excludedEndpoints` list, you can also prevent an endpoint from being generated by setting its
  value to `false`. All other endpoints will be generated. The complete set of configurable
  properties on these endpoints is described in the [Endpoint Configuration](#endpoint-configuration)
  section below. Here is a list of all available endpoints, including their corresponding HTTP method,
  path, and a short description of their behavior:
  - `getAll` [_Endpoint_](#endpoint-configuration)
    - `GET /api/collection`
    - Return a list of all entities within the collection (filtered searching via query params
      coming soon!).
  - `deleteAll` _Endpoint_
    - `DELETE /api/collection`
    - Remove all entities in the collection. **Be very careful with this endpoint!** This should
      probably only be used for development or by users with a specific role, like 'admin'.
  - `post` _Endpoint_
    - `POST /api/collection`
    - Add a new entity to the collection. All data passed in the request body will be copied into
      the newly created entity. **Warning: This is unsafe for now, as no type or bounds checking is
      done.**
  - `get` _Endpoint_
    - `GET /api/collection/:id`
    - Return the entity with the given `:id`.
  - `put` _Endpoint_
    - `PUT /api/collection/:id`
    - Completely replace the entity with the given `:id` with the data contained in the request
      body. Any fields not included will be removed from the document in the collection.
  - `delete` _Endpoint_
    - `DELETE /api/collection/:id`
    - Remove the entity with the given `:id` from the collection.

#### Endpoint Configuration

By default, each of the endpoints listed above is `undefined`, which means it will be generated with
any default route options. If you need finer control over your endpoints, each can be defined as an
object containing the following properties:

##### `authRequired`
- _Boolean_
- Default: `undefined`
- If true, this endpoint will return a `401` if the user is not properly [authenticated]
  (#authenticating). If defined, this overrides the option of the same name defined on the entire
  route.

##### `roleRequired`
- _String or Array of Strings_
- Default: `undefined` (no role required)
- The acceptable user roles for this endpoint (e.g.,
  `'admin'`, `['admin', 'dev']`). These roles will be accepted in addition to any defined over the
  entire route. If the authenticated user does not belong to at least one of the accepted roles, a
  `401` is returned. Since a role cannot be verified without an authenticated user, setting the
  `roleRequired` implies `authRequired: true`, so that option can be omitted without any
  consequence. For more on setting up roles, check out the [`alanning:roles`][alanning-roles]
  package.

##### `action`
- _Function_
- Default: `undefined` (Default endpoint generated)
- If you need to completely override the default endpoint behavior, you can provide a function
  that will be executed when the corresponding request is made. No parameters are passed; instead,
  `this` contains the [endpoint context](#endpoint-context), with properties including the URL and
  query parameters.


### Request and Response Structure

All responses generated by Restivus follow the [JSend] format, with one minor tweak: failures have
an identical structure to errors. Sample requests and responses for each endpoint are included
below:

#### `post`
Request:
```bash
curl -X POST http://localhost:3000/api/posts/ -d "title=Witty Title" -d "author=Jack Rose"
```

Response:
```json
{
  "status": "success",
  "data": {
    "_id": "LrcEYNojn5N7NPRdo",
    "title": "Witty Title",
    "author": "Jack Rose"
  }
}
```

#### `getAll`
Request:
```bash
curl -X GET http://localhost:3000/api/posts/
```

Response:
```json
{
  "status": "success",
  "data": [
    {
      "_id": "LrcEYNojn5N7NPRdo",
      "title": "Witty Title!",
      "author": "Jack Rose",
    },
    {
      "_id": "7F89EFivTnAcPMcY5",
      "title": "Average Stuff",
      "author": "Joe Schmoe",
    }
  ]
}
```

#### `deleteAll`
Request:
```bash
curl -X DELETE http://localhost:3000/api/posts/
```

Response:
```json
{
  "status": "success",
  "data": {
    "message": "Removed 2 items"
  }
}
```

#### `get`
Request:
```bash
curl -X GET http://localhost:3000/api/posts/LrcEYNojn5N7NPRdo
```

Response:
```json
{
  "status": "success",
  "data": {
    "_id": "LrcEYNojn5N7NPRdo",
    "title": "Witty Title",
    "author": "Jack Rose",
  }
}
```

#### `put`
Request:
```bash
curl -X PUT http://localhost:3000/api/posts/LrcEYNojn5N7NPRdo -d "title=Wittier Title" -d "author=Jaclyn Rose"
```

Response:
```json
{
  "status": "success",
  "data": {
    "_id": "LrcEYNojn5N7NPRdo",
    "title": "Wittier Title",
    "author": "Jaclyn Rose"
  }
}
```

#### `delete`
Request:
```bash
curl -X DELETE http://localhost:3000/api/posts/LrcEYNojn5N7NPRdo
```

Response:
```json
{
  "status": "success",
  "data": {
    "message": "Item removed"
  }
}
```

### Users Collection Endpoints

A few special exceptions have been made for routes added for the `Meteor.users` collection. For now,
the majority of the operations are limited to read access to the `user._id` and read/write access to
the `user.profile`. All route and endpoint options are identical to those described for all other
collections above. No options have been configured in the examples below; however, it is highly
recommended that role permissions be setup (or at the absolute least, authentication required) for
the `delete` and `deleteAll` endpoints. Below are sample requests and responses for the users
collection.

Create collection:
```javascript
Restivus.addCollection(Meteor.users);
```

#### `post`
Request:
`POST http://localhost:3000/api/users`
```json
{
  "email": "jack@mail.com",
  "password": "password",
  "profile": {
    "firstName": "Jack",
    "lastName": "Rose"
  }
}
```
_Note: The only fields that will be recognized in the request body when creating a new user are
`email`, `username`, `password`, and `profile`. These map directly to the parameters of the same
name in the [Accounts.createUser()](http://docs.meteor.com/#/full/accounts_createuser) method, so
check that out for more information on how those fields are handled._

Response:
```json
{
  "status": "success",
  "data": {
    "_id": "oFpdgAMMr7F5A7P3a",
    "profile": {
      "firstName": "Jack",
      "lastName": "Rose"
    }
  }
}
```

#### `getAll`
Request:
```bash
curl -X GET http://localhost:3000/api/users/
```

Response:
```json
{
  "status": "success",
  "data": [
    {
      "_id": "nBTnv83sTrf38fFTi",
      "profile": {
        "firstName": "Anthony",
        "lastName": "Reid"
      }
    },
    {
      "_id": "oFpdgAMMr7F5A7P3a",
      "profile": {
        "firstName": "Jack",
        "lastName": "Rose"
      }
    }
  ]
}
```


#### `deleteAll`
Request:
```bash
curl -X DELETE http://localhost:3000/api/users/
```

Response:
```json
{
  "status": "success",
  "data": {
    "message": "Removed 2 users"
  }
}
```

#### `get`
Request:
```bash
curl -X GET http://localhost:3000/api/users/oFpdgAMMr7F5A7P3a
```

Response:
```json
{
  "status": "success",
  "data": {
    "_id": "oFpdgAMMr7F5A7P3a",
    "profile": {
      "firstName": "Jack",
      "lastName": "Rose"
    }
  }
}
```

#### `put`
Request:
`PUT http://localhost:3000/api/users/oFpdgAMMr7F5A7P3a`
```json
{
    "firstName": "Jaclyn",
    "age": 25
}
```
_Note: The data included in the request body will completely overwrite the `user.profile` field of
the User document_

Response:
```json
{
  "status": "success",
  "data": {
    "_id": "oFpdgAMMr7F5A7P3a",
    "profile": {
      "firstName": "Jaclyn",
      "age": "25"
    }
  }
}
```

#### `delete`
Request:
```bash
curl -X DELETE http://localhost:3000/api/users/oFpdgAMMr7F5A7P3a
```
Response:
```json
{
  "status": "success",
  "data": {
    "message": "User removed"
  }
}
```


## Defining Custom Routes

Routes are defined using `Restivus.addRoute()`. A route consists of a path and a set of endpoints
defined at that path.

### Path Structure

The `path` is the 1st parameter of `Restivus.addRoute`. You can pass it a string or regex. If you
pass it `test/path`, the full path will be `https://yoursite.com/api/test/path`.

Paths can have variable parameters. For example, you can create a route to show a post with a
specific id. The `id` is variable depending on the post you want to see such as "/posts/1" or
"/posts/2". To declare a named parameter in the path, use the `:` syntax followed by the parameter
name. When a user goes to that url, the actual value of the parameter will be stored as a property
on `this.urlParams` in your endpoint function.

In this example we have a parameter named `_id`. If we navigate to the `/post/5` url in our browser,
inside of the GET endpoint function we can get the actual value of the `_id` from
`this.urlParams._id`. In this case `this.urlParams._id => 5`.

###### CoffeeScript:
```coffeescript
# Given a url like "/post/5"
Restivus.addRoute '/post/:_id',
  get: ->
    id = @urlParams._id # "5"
```
###### JavaScript:
```javascript
// Given a url "/post/5"
Restivus.addRoute('/post/:_id', {
  get: function () {
    var id = this.urlParams._id; // "5"
  }
});
```

You can have multiple URL parameters. In this example, we have an `_id` parameter and a `commentId`
parameter. If you navigate to the url `/post/5/comments/100` then inside your endpoint function
`this.urlParams._id => 5` and `this.urlParams.commentId => 100`.

###### CoffeeScript:
```coffeescript
# Given a url "/post/5/comments/100"
Restivus.addRoute '/post/:_id/comments/:commentId',
  get: ->
    id = @urlParams._id # "5"
    commentId = @urlParams.commentId # "100"
```

###### JavaScript:
```javascript
// Given a url "/post/5/comments/100"
Restivus.addRoute('/post/:_id/comments/:commentId', {
  get: function () {
    var id = this.urlParams._id; // "5"
    var commentId = this.urlParams.commentId; // "100"
  }
});
```

If there is a query string in the url, you can access that using `this.queryParams`.

###### Coffeescript:
```coffeescript
# Given the url: "/post/5?q=liked#hash_fragment"
Restivus.addRoute '/post/:_id',
  get: ->
    id = @urlParams._id
    query = @queryParams # query.q -> "liked"
```

###### JavaScript:
```javascript
// Given the url: "/post/5?q=liked#hash_fragment"
Restivus.addRoute('/post/:_id', {
  get: function () {
    var id = this.urlParams._id;
    var query = this.queryParams; // query.q -> "liked"
  }
});
```

### Route Options

The following options are available in Restivus.addRoute (as the 2nd, optional parameter):
##### `authRequired`
- _Boolean_
- Default: `false`
- If true, all endpoints on this route will return a `401` if the user is not properly
  [authenticated](#authenticating).

##### `roleRequired`
- _String or Array of Strings_
- Default: `undefined` (no role required)
- A string or array of strings corresponding to the acceptable user roles for all endpoints on
  this route (e.g., `'admin'`, `['admin', 'dev']`). Additional role permissions can be defined on
  specific endpoints. If the authenticated user does not belong to at least one of the accepted
  roles, a `401` is returned. Since a role cannot be verified without an authenticated user,
  setting the `roleRequired` implies `authRequired: true`, so that option can be omitted without
  any consequence. For more on setting up roles, check out the [`alanning:roles`][alanning-roles]
  package.

### Defining Endpoints

The last parameter of Restivus.addRoute is an object with properties corresponding to the supported
HTTP methods. At least one method must have an endpoint defined on it. The following endpoints can
be defined in Restivus:
- `get`
- `post`
- `put`
- `patch`
- `delete`
- `options`

These endpoints can be defined one of two ways. First, you can simply provide a function for each
method you want to support at the given path. The corresponding endpoint will be executed when that
type of request is made at that path.

For finer-grained control over each endpoint, you can also define each one as an object
containing the endpoint action and some addtional configuration options. 

#### Endpoint Configuration

An `action` is required when configuring an endpoint. All other configuration settings are optional, 
and will get their default values from the route.

##### `action`
- _Function_
- Default: `undefined`
- A function that will be executed when a request is made for the corresponding HTTP method.

##### `authRequired`
- _String_
- Default: [`Route.authRequired`](#authrequired-1)
- If true, this endpoint will return a `401` if the user is not properly
  [authenticated](#authenticating). Overrides the option of the same name defined on the entire
  route.

##### `roleRequired`
- _String or Array of Strings_
- Default: [`Route.roleRequired`](#rolerequired-1)
- The acceptable user roles for this endpoint (e.g.,
  `'admin'`, `['admin', 'dev']`). These roles will be accepted in addition to any defined over the
  entire route. If the authenticated user does not belong to at least one of the accepted roles, a
  `401` is returned. Since a role cannot be verified without an authenticated user, setting the
  `roleRequired` implies `authRequired: true`, so that option can be omitted without any
  consequence. For more on setting up roles, check out the [`alanning:roles`][alanning-roles]
  package.

###### CoffeeScript
```coffeescript
Restivus.addRoute 'posts', {authRequired: true},
  get:
    authRequired: false
    action: ->
      # GET api/posts
  post: ->
    # POST api/posts
  put: ->
    # PUT api/posts
  patch: ->
    # PATCH api/posts
  delete: ->
    # DELETE api/posts
  options: ->
    # OPTIONS api/posts
```

###### JavaScript
```javascript
Restivus.addRoute('posts', {authRequired: true}, {
  get: function () {
    authRequired: false
    action: function () {
      // GET api/posts
    }
  },
  post: function () {
    // POST api/posts
  },
  put: function () {
    // PUT api/posts
  },
  patch: function () {
    // PATCH api/posts
  },
  delete: function () {
    // DELETE api/posts
  },
  options: function () {
    // OPTIONS api/posts
  }
});
```
In the above examples, all the endpoints except the GETs will require [authentication]
(#authenticating).

#### Endpoint Context

Each endpoint has access to:

##### `this.user`
- _Meteor.user_
- The authenticated `Meteor.user`. Only available if `useAuth` and `authRequired` are both `true`.
  If not, it will be `undefined`.

##### `this.userId`
- _String_
- The authenticated user's `Meteor.userId`. Only available if `useAuth` and `authRequired` are both
  `true`. If not, it will be `undefined`.

##### `this.urlParams`
- _Object_
- Non-optional parameters extracted from the URL. A parameter `id` on the path `posts/:id` would be
  available as `this.urlParams.id`.

##### `this.queryParams`
- _Object_
- Optional query parameters from the URL. Given the url `https://yoursite.com/posts?likes=true`,
  `this.queryParams.likes => true`.

##### `this.bodyParams`
- _Object_
- Parameters passed in the request body. Given the request body `{ "friend": { "name": "Jack" } }`,
  `this.bodyParams.friend.name => "Jack"`.

##### `this.request`
- [_Node request object_][node-request]

##### `this.response`
- [_Node response object_][node-response]
- If you handle the response yourself using `this.response.write()` or `this.response.writeHead()`
  you **must** call `this.done()`. In addition to preventing the default response (which will throw
  an error if you've initiated the response yourself), it will also close the connection using
  `this.response.end()`, so you can safely omit that from your endpoint.

##### `this.done()`
- _Function_
- **Must** be called after handling the response manually with `this.response.write()` or
  `this.response.writeHead()`. This must be called immediately before returning from an endpoint.

  ```javascript
  Restivus.addRoute('manualResponse', {
    get: function () {
      console.log('Testing manual response');
      this.response.write('This is a manual response');
      this.done();  // Must call this immediately before return!
    }
  });
  ```
  
##### `this.<endpointOption>`
All [endpoint configuration options](#endpoint-configuration-1) can be accessed by name (e.g., 
`this.roleRequired`). Within an endpoint, all options have been completely resolved, meaning all 
configuration options set on an endpoint's route will already be applied to the endpoint as
defaults. So if you set `authRequired: true` on a route and do not set the `authRequired` option on 
one if its endpoints, `this.authRequired` will still be `true` within that endpoint, since the 
default will already have been applied from the route. 

#### Response Data

You can return a raw string:
```javascript
return "That's current!";
```

A JSON object:
```javascript
return { json: 'object' };
```

A raw array:
```javascript
return [ 'red', 'green', 'blue' ];
```

Or include a `statusCode` or `headers`. At least one must be provided along with the `body`:
```javascript
return {
  statusCode: 404,
  headers: {
    'Content-Type': 'text/plain',
    'X-Custom-Header': 'custom value'
  },
  body: 'There is nothing here!'
};
```

All responses contain the following defaults, which will be overridden with any provided values:

##### statusCode
- Default: `200`

##### headers
- Default:
  - `Content-Type`: `application/json`
  - `Access-Control-Allow-Origin`: `*`
    - This is a [CORS-compliant header][cors] that allows requests to be made to the API from any 
      domain. Without this, requests from within the browser would only be allowed from the same 
      domain the API is hosted on, which is typically not the intended behavior. This can be 
      [disabled by default](https://github.com/kahmali/meteor-restivus#enablecors), or also by 
      returning a header of the same name with a domain specified (usually the domain the API is 
      being hosted on).


## Documenting Your API

What's a REST API without awesome docs? I'll tell you: absolutely freaking useless. So to fix that,
we use and recommend [apiDoc][]. It allows you to generate beautiful and extremely handy API docs
from your JavaScript or CoffeeScript comments. It supports other comment styles as well, but we're
Meteorites, so who cares? Check it out. Use it.

# Consuming A Restivus API

The following uses the above code.

## Basic Usage

We can call our `POST /posts/:id/comments` endpoint the following way. Note the /api/ in the URL
(defined with the api_path option above):
```bash
curl -d "message=Some message details" http://localhost:3000/api/posts/3/comments
```

## Authenticating

If you have `useAuth` set to `true`, you now have a `POST /api/login` endpoint that returns a
`userId` and `authToken`. You must save these, and include them in subsequent requests.

**Warning: Make sure you're using HTTPS, otherwise this is insecure! In an ideal world, this should
only be done with DDP and SRP, but, alas, this is a REST API.**

```bash
curl http://localhost:3000/api/login/ -d "password=testpassword&user=test"
```

The response will look like this, which you must save (for subsequent authenticated requests):
```javascript
{ status: "success", data: {authToken: "f2KpRW7KeN9aPmjSZ", userId: fbdpsNf4oHiX79vMJ} }
```
  
You also have an authenticated `GET /api/logout` endpoint for logging a user out. If successful, the
auth token that is passed in the request header will be invalidated (removed from the user account),
so it will not work in any subsequent requests.
```bash
curl http://localhost:3000/api/logout -H "X-Auth-Token: f2KpRW7KeN9aPmjSZ" -H "X-User-Id: fbdpsNf4oHiX79vMJ"
```

## Authenticated Calls

For any endpoints that require the default authentication, you must include the `userId` and
`authToken` with each request under the following headers:
- X-User-Id
- X-Auth-Token

```bash
curl -H "X-Auth-Token: f2KpRW7KeN9aPmjSZ" -H "X-User-Id: fbdpsNf4oHiX79vMJ" http://localhost:3000/api/posts/
```

# Resources

## Change Log

A detailed list of the changes between versions can be found in the [change log]
(https://github.com/kahmali/meteor-restivus/blob/master/CHANGELOG.md).

## Contributing

Contributions to Restivus are welcome and appreciated! If you're interested in contributing, please
check out [the guidelines](https://github.com/kahmali/meteor-restivus/blob/master/CONTRIBUTING.md)
before getting started.

## Thanks

Thanks to the developers over at Differential for [RestStop2][], where we got our inspiration for
this package and stole tons of ideas and code, as well as the [Iron Router][iron-router] team for
giving us a solid foundation with their server-side routing in Meteor.

Also, thanks to the following projects, which RestStop2 was inspired by:
- [gkoberger/meteor-reststop](https://github.com/gkoberger/meteor-reststop)
- [tmeasday/meteor-router](https://github.com/tmeasday/meteor-router)
- [crazytoad/meteor-collectionapi](https://github.com/crazytoad/meteor-collectionapi)

## License

MIT License. See [LICENSE](https://github.com/kahmali/meteor-restivus/blob/master/LICENSE) for
details.


[restivus-issues]:      https://github.com/kahmali/meteor-restivus/issues                      "Restivus Issues"
[restivus-change-log]:  https://github.com/kahmali/meteor-restivus/blob/master/CHANGELOG.md    "Restivus Change Log"
[reststop2-docs]:       http://github.differential.com/reststop2/                              "RestStop2 Docs"
[reststop2]:            https://github.com/Differential/reststop2                              "RestStop2"
[iron-router]:          https://github.com/EventedMind/iron-router                             "Iron Router"
[node-request]:         http://nodejs.org/api/http.html#http_http_incomingmessage              "Node Request Object Docs"
[node-response]:        http://nodejs.org/api/http.html#http_class_http_serverresponse         "Node Response Object Docs"
[jsend]:                http://labs.omniti.com/labs/jsend                                      "JSend REST API Standard"
[apidoc]:               http://apidocjs.com/                                                   "apiDoc"
[alanning-roles]:       https://github.com/alanning/meteor-roles                               "Meteor Roles Package"
[cors]:                 https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS  "Cross Origin Resource Sharing (CORS)"
