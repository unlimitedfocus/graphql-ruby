---
title:  Relay — Mutations
---

Relay uses a [strict mutation API](https://facebook.github.io/relay/docs/graphql-mutations.html#content) for modifying the state of your application. This API makes mutations predictable to the client.


## Mutation root

To add mutations to your GraphQL schema, define a mutation type and pass it to your schema:

```ruby
# Define the mutation type
MutationType = GraphQL::ObjectType.define do
  name "Mutation"
  # ...
end

# and pass it to the schema
MySchema = GraphQL::Schema.define do
  query QueryType
  mutation MutationType
end
```

Like `QueryType`, `MutationType` is a root of the schema.

## Mutation fields

Members of `MutationType` are _mutation fields_. For GraphQL in general, mutation fields are identical to query fields _except_ that they have side-effects (which mutate application state, eg, update the database).

For Relay-compliant GraphQL, a mutation field must comply to a strict API. `GraphQL::Relay` includes a mutation definition helper (see below) to make it simple.

After defining a mutation (see below), add it to your mutation type:

```ruby
MutationType = GraphQL::ObjectType.define do
  name "Mutation"
  # Add the mutation's derived field to the mutation type
  field :addComment, field: AddCommentMutation.field
  # ...
end
```

## Relay mutations

To define a mutation, use `GraphQL::Relay::Mutation.define`. Inside the block, you should configure:
  - `name`, which will name the mutation field & derived types
  - `input_field`s, which will be applied to the derived `InputObjectType`
  - `return_field`s, which will be applied to the derived `ObjectType`
  - `resolve(->(object, inputs, ctx) { ... })`, the mutation which will actually happen

For example:

```ruby
AddCommentMutation = GraphQL::Relay::Mutation.define do
  # Used to name derived types, eg `"AddCommentInput"`:
  name "AddComment"

  # Accessible from `input` in the resolve function:
  input_field :postId, !types.ID
  input_field :authorId, !types.ID
  input_field :content, !types.String

  # The result has access to these fields,
  # resolve must return a hash with these keys
  return_field :post, PostType
  return_field :comment, CommentType

  # The resolve proc is where you alter the system state.
  resolve ->(object, inputs, ctx) {
    post = Post.find(inputs[:postId])
    comment = post.comments.create!(author_id: inputs[:authorId], content: inputs[:content])

    # The keys in this hash correspond to `return_field`s above:
    {
      comment: comment,
      post: post,
    }
  }
end
```

## Derived Objects

`graphql-ruby` uses your mutation to define some members of the schema. Under the hood, GraphQL creates:

- A field for your schema's `mutation` root, as `AddCommentMutation.field`
- A derived `InputObjectType` for input values, as `AddCommentMutation.input_type`
- A derived `ObjectType` for return values, as `AddCommentMutation.return_type`

Each of these derived objects maintains a reference to the parent `Mutation` in the `mutation` attribute. So you can access it from the derived object:

```ruby
# Define a mutation:
AddCommentMutation = GraphQL::Relay::Mutation.define { ... }
# Get the derived input type:
AddCommentMutationInput = AddCommentMutation.input_type
# Reference the parent mutation:
AddCommentMutationInput.mutation
# => #<GraphQL::Relay::Mutation @name="AddComment">
```

## Mutation Resolution

In the mutation's `resolve` function, it can mutate your application state (eg, writing to the database) and return some results.

`resolve` is called with:

- `object`, which is the `root_value:` provided to `Schema.execute`
- `inputs`, which is a hash whose keys are the ones defined by `input_field`. (This value comes from `args[:input]`.)
- `ctx`, which is the query context

It must return a `hash` whose keys match your defined `return_field`s. (Or, if you specified a `return_type`, you can return an object suitable for that type.)

### Specify a Return Type

Instead of specifying `return_field`s, you can specify a `return_type` for a mutation. This type will be used to expose the object returned from `resolve`.

```ruby
CreateUser = GraphQL::Relay::Mutation.define do
  return_type UserMutationResultType
  # ...
  resolve ->(obj, input, ctx) {
    user = User.create(input)
    # this object will be treated as `UserMutationResultType`
    UserMutationResult.new(user, client_mutation_id: input[:clientMutationId])
  }
end
```

If you provide your own return type, it's up to you to support `clientMutationId`