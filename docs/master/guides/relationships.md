# Eloquent Relationships

Eloquent relationships can be accessed just like any other properties.
This makes it super easy to use in your schema.

Suppose you have defined the following model:

```php
<?php

namespace App;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Post extends Model
{
    public function comments(): HasMany
    {
        return $this->hasMany(Comment::class);
    }
    
    public function author(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
```

Just add fields to your type that are named just like the relationships:

```graphql
type Post {
  author: User
  comments: [Comment!]
}
```

This approach is fine if performance is not super critical or if you only fetch a single post.
However, as your queries become larger and more complex, you might want to optimize performance.

## Querying Relationships

Just like in Laravel, you can define [Eloquent Relationships](https://laravel.com/docs/eloquent-relationships) in your schema.
Lighthouse has got you covered with specialized directives that optimize the Queries for you.

Suppose you want to load a list of posts and associated comments. When you tell
Lighthouse about the relationship, it automatically eager loads the comments when you need them.

For special cases, you can use [`@with`](../api-reference/directives.md#with) to eager-load a relation
without returning it directly.

### One To One

Use the [@hasOne](../api-reference/directives.md#hasone) directive to define a [one-to-one relationship](https://laravel.com/docs/eloquent-relationships#one-to-one)
between two types in your schema.

```graphql
type User {
  phone: Phone @hasOne
}
```

The inverse can be defined through the [@belongsTo](../api-reference/directives.md#belongsto) directive.

```graphql
type Phone {
  user: User @belongsTo
}
```

### One To Many

Use the [@hasMany](../api-reference/directives.md#hasmany) directive to define a [one-to-many relationship](https://laravel.com/docs/eloquent-relationships#one-to-many).

```graphql
type Post {
  comments: [Comment!]! @hasMany
}
```

Again, the inverse is defined with the [@belongsTo](../api-reference/directives.md#belongsto) directive.

```graphql
type Comment {
  post: Post! @belongsTo
}
```

### Many To Many

While [many-to-many relationships](https://laravel.com/docs/5.7/eloquent-relationships#many-to-many)
are a bit more work to set up in Laravel, defining them in Lighthouse is a breeze.
Use the [@belongsToMany](../api-reference/directives.md#belongstomany) directive to define it.

```graphql
type User {
  roles: [Role!]! @belongsToMany
}
```

The inverse works the same.

```graphql
type Role {
  users: [User!]! @belongsToMany
}
```

## Renaming Relations

When you define a relation, Lighthouse assumes that the field and the relationship
method have the same name. If you need to name your field differently, you have to
specify the name of the method.

```
type Post {
  author: User! @belongsTo(relation: "user")
}
```

This would work for the following model:

```php
<?php

namespace App;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Post extends Model 
{
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
```

## Mutating Relationships

Lighthouse allows you to create, update or delete your relationships in
a single mutation.

You have to define return types on your relationship methods so that Lighthouse
can detect them.

```php
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Post extends Model 
{
    // WORKS
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    // DOES NOT WORK
    public function comments()
    {
        return $this->hasMany(Comment::class);        
    }
}
```

By default, all mutations are wrapped in a database transaction.
If any of the nested operations fail, the whole mutation is aborted
and no changes are written to the database.
You can change this setting [in the configuration](../getting-started/configuration.md).

### Belongs To

We will start of by defining a mutation to create a post.

```graphql
type Mutation {
  createPost(input: CreatePostInput!): Post @create(flatten: true)
}
```

The mutation takes a single argument `input` that contains data about
the Post you want to create.

```graphql
input CreatePostInput {
  title: String!
  author: CreateAuthorRelation
}
```

The first argument `title` is a value of the `Post` itself and corresponds
to a column in the database.

The second argument `author`, exposes operations on the related `User` model.
It has to be named just like the relationship method that is defined on the `Post` model.

```graphql
input CreateAuthorRelation {
  connect: ID
  create: CreateUserInput
}
```

There are two possible operations that you can expose on a `BelongsTo` relationship:
- `connect` it to an existing model
- `create` and attach a new related model

Finally, you need to define the input that allows you to create a new `User`.

```graphql
input CreateUserInput {
  name: String!
}
```

To create a new model and connect it to an existing model,
just pass the ID of the model you want to associate.

```graphql
mutation {
  createPost(input: {
    title: "My new Post"
    author: {
      connect: 123
    }
  }){
    id
    author {
      name
    }
  }
}
```

Lighthouse will create a new `Post` and associate an `User` with it.

```json
{
  "data": {
    "createPost": {
      "id": 456,
      "author": {
        "name": "Herbert"
      }
    }
  }
}
```

If the related model does not exist yet, you can also
create a new one.

```graphql
mutation {
  createPost(input: {
    title: "My new Post"
    author: {
      create: {
        name: "Gina"
      }  
    }
  }){
    id
    author {
      id
    }
  }
}
```

```json
{
  "data": {
    "createPost": {
      "id": 456,
      "author": {
        "id": 55
      }
    }
  }
}
```

You may also allow the user to change or remove a relation.

```graphql
type Mutation {
  updatePost(input: UpdatePostInput!): Post @update(flatten: true)
}

input UpdatePostInput {
  title: String
  author: ID
}
```

If you want to remove a relation, simply set it to `null`,

```graphql
mutation {
  updatePost(input: {
    title: "An updated title"
    author: null
  }){
    title
    author {
      name
    }
  }
}
```

```json
{
  "data": {
    "updatePost": {
      "title": "An updated title",
      "author": null
    }
  }
}
```

### Has Many

The counterpart to a `BelongsTo` relationship is `HasMany`. We will start
of by defining a mutation to create an `User`.

```graphql
type Mutation {
  createUser(input: CreateUserInput!): User @create(flatten: true)
}
```

This mutation takes a single argument `input` that contains values
of the `User` itself and its associated `Post` models.

```graphql
input CreateUserInput {
  name: String!
  posts: CreatePostsRelation
}
```

Now, we can an operation that allows us to directly create new posts
right when we create the `User`.

```graphql
input CreatePostsRelation {
  create: [CreatePostInput!]!
}

input CreatePostInput {
  title: String!
}
```

You can now create a `User` and some posts with it in one request.

```graphql
mutation {
  createUser(input: {
    name: "Phil"
    posts: {
      create: [
        {
          title: "Phils first post"
        },
        {
          title: "Awesome second post"
        }
      ]  
    }
  }){
    id
    posts {
      id
    }
  }
}
```

```json
{
  "data": {
    "createUser": {
      "id": 23,
      "posts": [
        {
          "id": 434
        },
        {
          "id": 435
        }
      ]
    }
  }
}
```

### Belongs To Many


```graphql
type Mutation {
  createPost(input: CreatePostInput!): Post @create(flatten: true)
}

input CreateAuthorRelation {
  connect: [ID]
  create: [CreateAuthorInput]
}

input CreateAuthorInput {
  name: String!
}

input CreatePostInput {
  title: String!
  authors: CreateAuthorRelation
}
```

Just pass the ID of the models you want to associate.

```graphql
mutation {
  createPost(input: {
    title: "My new Post"
    authors: {
      connect: [123,124]
    }
  }){
    id
    authors {
      name
    }
  }
}
```

Or create a new ones.

```graphql
mutation {
  createPost(input: {
    title: "My new Post"
    author: {
      create: [{
        name: "Herbert"
      },
      {
        name: "Bro"
      }]  
    }
  }){
    id
    authors {
      name
    }
  }
}
```

Lighthouse will detect the relationship and attach/create it.

```json
{
  "data": {
    "createPost": {
      "id": 456,
      "authors": [{
        "name": "Herbert"
      },
      {
        "name": "Bro"
      }]
    }
  }
}
```

### MorphTo

```graphql
type Task {
  id: ID
  name: String
  hour: Hour
}

type Hour {
  id: ID
  weekday: Int
  hourable: Task
}

type Mutation {
  createHour(input: CreateHourInput!): Hour @create(flatten: true)
}

input CreateHourInput {
  hourable_type: String!
  hourable_id: Int!
  from: String
  to: String
  weekday: Int
}
```

```graphql
mutation {
  createHour(input: {
      hourable_type: "App\\\Task"
      hourable_id: 1
      weekday: 2
  }) {
      id
      weekday
      hourable {
          id
          name
      }
  }
}
```
