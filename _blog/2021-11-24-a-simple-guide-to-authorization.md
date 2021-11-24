---
excerpt: "Authorization is something that's usually an afterthought. In this article we'll look at how we can do it properly."
title: "A simple guide to authorization"
tags: [authorization, security]
author: addamsson
short_title: "A simple guide to authorization"
series: content-gateway
comments: true
updated_at: 2021-11-24
---

> In this article we'll look at how to create an effective authorization system. This is only a high level overview, but it's a good starting point if you want to get started with this topic.

I've worked at many places and there is one problem that often comes up, but it is usually not a primary concern. This is **authorization**. According to the latest [OWASP Top 10](https://owasp.org/www-project-top-ten/) broken access control is the most common problem.

Unfortunately on most platforms there are no good libraries that try to tackle this problem, and even if there are solutions for it, they are usually not centralized and using them will litter your codebase with this cross-cutting concern:

```java
@PreAuthorize("hasRole('USER')")
public void create(Contact contact);
```

What happens if I have more complex criteria? How can I encode it properly? What if I want to write custom code? What happens in practice is that this is usually a part of the business logic. I'm going to try to give you some hints on what to do about this.

## Authorization Models

A common misconception about authorization is that it is a synonym of authentication, but this is not the case. **Authentication** is the process of verifying a user's identity and credentials. **Authorization** is the process of verifying that the user has the right to perform a certain action. What usually happens is that we first authenticate the user, and when we know their identity, we can authorize them.

With this out of the way let's take a look at what authorization models are out there.

### Access Control List

This is the simplest possible authorization model you can use. It is just a list of users who are authorized to do... well, everything. Example:

**A bouncer in an exclusive club.** They have a list of people they are allowed to let in, and that's it.

### Role Based Access Control

This is a little more fine grained than the ACL. Here every user has a *role* and what they can do is based on that role. For example the java code example above was an implementation of the RBAC model. With RBAC you can have roles like `["ADMIN", "USER"]` and so on, and then you can annotate your code with things like `@PreAuthorize`. Pretty simple.

What you can add on top is to bind *roles* to *permissions*. For example you can have something like this:

```json
{
    "roles": {
        "admin": {
            "permissions": [
                "view_bounties",
                "view_bounty",
                "create_bounty",
                "publish_draft_bounty",
                "share_draft_bounty",
                "edit_draft_bounty",
                "unpublish_bounty",
                "delete_bounty"
            ]
        },
        "user": {
            "permissions": [
                "view_bounties",
                "view_bounty",
                "create_bounty",
                "publish_draft_bounty",
                "share_draft_bounty",
                "edit_draft_bounty",
                "delete_bounty"
            ]
        },
        "anonymous": {
            "permissions": ["view_bounties", "view_bounty"]
        }
    }
}
```

The problem with RBAC is that it doesn't allow for fine-grained access control. We'll see soon how this can become a problem. An example of RBAC would be:

**A bouncer in an exclusive club** who gives out wristbands based on what he has on his list. Then:

- If you have a red wristband you can drink in the club
- If you have an orange one you can play roulette

> Note that you usually get one wristband, not multiple, but I'm trying to keep it simple.

### Policy Based Access Control

Let's say that we want to allow everybody to view our *bounties*, but we only want to allow *admins* to view *all* of them. Regular users should only be able to see *published* ones. How can we do that? Enter **policies**.

A **policy** is something that controls access to a function based on some attribute. For example you only want to allow `user`s to delete their own bounties, but for admins you want to enable deleting for all. This is something that you can't properly encode with just a list of permissions.

> Note that in the example below where there are no `policies` assume that there is the default `["allow_for_all"]` policy. This is important, because it is better to disable **everything** and selectively enable them back (whitelisting) instead of forbidding things (blacklist). Using blacklists is one of the biggest security issues that you can create for yourself.

```json
{
    "roles": {
        "admin": {
            "permissions": [
                { "name": "view_bounties" },
                { "name": "view_bounty" },
                { "name": "create_bounty" },
                { "name": "publish_draft_bounty" },
                { "name": "share_draft_bounty" },
                { "name": "edit_draft_bounty" },
                { "name": "unpublish_bounty" },
                { "name": "delete_bounty" }
            ]
        },
        "user": {
            "permissions": [
                {
                    "name": "view_bounties",
                    "policies": ["allow_for_published"]
                },
                { "name": "view_bounty", "policies": ["allow_for_published"] },
                { "name": "create_bounty" },
                {
                    "name": "publish_draft_bounty",
                    "policies": ["allow_for_owner"]
                },
                {
                    "name": "share_draft_bounty",
                    "policies": ["allow_for_owner"]
                },
                {
                    "name": "edit_draft_bounty",
                    "policies": ["allow_for_owner"]
                },
                { "name": "delete_bounty", "policies": ["allow_for_owner"] }
            ]
        },
        "anonymous": {
            "permissions": [
                {
                    "name": "view_bounties",
                    "policies": ["allow_for_published"]
                },
                { "name": "view_bounty", "policies": ["allow_for_published"] }
            ]
        }
    }
}
```

With a model like this you can fine-tune what to allow and when and you can attach functions to policies.

An example for this is when your **bouncer in the exclusive club** gives you the wristband, but they won't serve you in the bar if you're under `21`.


### Risk Adaptive-Based Access Control (RAdAC)

This model is similar to *PBAC* but it also takes the *context* into account. For example the **bouncer in the exclusive club** would give you the red wristband, you're over `21`, but you are not vaccinated so they won't let you in.

## A Possible Solution

Now that we know what authorization models are out there, let's see how we can implement one.

> I'm going to use Typescript for the examples. Some parts of the code are omitted for brevity. You can
> check the full code [here](https://github.com/BanklessDAO/content-gateway/tree/develop/libs/shared/util-auth) if you're interested.

What most guides suggest to do is to have some `authorization` object that one can use to check if a certain operation can be executed or not:

```js
if (!auth.can(user, Operations.CREATE_BOUNTY, context)) {
    throw new AuthorizationError("Can't do this, sorry");
}
```

While this might look like a good idea I don't think it is the best way to do it. My first issue with this is that the business logic **knows** about the authorization. It no longer adheres to the [Single-responsibility Principle](https://en.wikipedia.org/wiki/Single-responsibility_principle) and if the authorization logic changes, the business logic will need to be updated.

Another problem with this is that it is easy to forget to check the authorization leading to the most common security mistake (Broken Access Control).

Let's see how we can codify this. First, we're going to create a type that represents an *operation* (without authorization):

> Note that I'm using [fp-ts](https://github.com/gcanti/fp-ts) here, which is a library that helps with type-safe functional programming. Fore those who are not familiar with it: a `Task` is a function that returns a `Promise`, so it represents an asynchronous operation that can fail. An `Either` is an object that either contains a value or an error. This means that a `TaskEither` represents an asynchronous operation that can't fail (won't throw an exception).

```ts
import { ProgramError } from "@shared/util-dto";
import * as TE from "fp-ts/lib/TaskEither";

export type Operation<I, O> = (input: I) => TE.TaskEither<AuthorizationError, O>;
```

Each *operation* takes an arbitrary input and produces a `TaskEither` (an asynchronous operation that can't fail). Here the type parameters `I` and `O` represent the input and the output.

An example of one such opearation is finding a `Todo`:

```ts
export const findTodo = (id: number) => {
    const todo = todos[id];
    if (todo) {
        return TE.right(todo);
    } else {
        return TE.left(new TodoNotFoundError(id));
    }
};
```

As you can see there is no authorization information in this operation, but we need a way to represent an `Operation` that is *authorized* and can be safely called:

```ts
export interface AuthorizedOperationBrand {
    readonly _: unique symbol;
}

export type AuthorizedOperation<I, O> = (
    input: TE.TaskEither<AuthorizationError, Context<I>>
) => TE.TaskEither<AuthorizationError, Context<O>> & AuthorizedOperationBrand;
```

Here we create a **branded type**. This means that an `AuthorizedOperation` will need a **smart constructor** and we can't just cast an `Operation` to an `AuthorizedOperation`.

Thiss type also represents a function, but instead of taking `I`, it takes a `TaskEither` that has a `Context` of `I`. We'll see soon why this is the case. As for the `Context`, it represents the context of the operation (this was mentioned in the *RAdAC* model above):

```ts
export interface Context<I> {
    user: AnyUser;
    data: I;
}
```

Right now we want to keep it simple, so a `Context` only has the `data` (this will be the input of the `Operation`) and a `user`.

`User` in our case is an entity, that has an `id`, a `name` and some `roles`:

```ts
export interface User<ID extends number | string> {
    id: ID;
    name: string;
    roles: string[];
}

export type AnyUser = User<any>;
```

For simplicity we only have the names of the roles stored in an array.

Now let's talk a bit about how an actual `Role` looks like:

```ts
export interface Role {
    name: string;
    permissions: AnyPermission[];
}
```

It is an object that has a name, and a list of `Permission`s:

```ts
export interface Permission<I, O> {
    name: string;
    operation: Operation<I, O>;
    policies: Policy<I>[];
    filters?: Filter<O>[];
}

export type AnyPermission = Permission<any, any>;
```

This should be familiar from the *PBAC* model, but we have a twist: `Policy` objects check the *context* of the operation:

```ts
export type Policy<I> = (
    context: Context<I>
) => TE.TaskEither<AuthorizationError, Context<I>>;
```

They can make a decision based on what's visible from the input and prevent the operation form being executed in the first place.

`Filter`s on the other hand:

```ts
export type Filter<O> = (
    context: Context<O>
) => TE.TaskEither<AuthorizationError, Context<O>>;
```

operate on the output of the operation. Both *policies* and *filters* can return an `AuthorizationError` in case something is not allowed.

We also include the `Operation` that's bound to the `Permission`.

### Authorizing an Operation

Now that we have most of the pieces in place let's see how the actual *authorization* can be performed. For this we're going to need an object that contains the authorization data:

```ts
export interface Authorization {
    roles: {
        [key: string]: Role;
    };
}
```

For simplicity's sake every `Permission` must belong in a `Role` and the `Authorization` object itself is just a mapping between the role names and the `Role` objects. The `authorize` operation:

```ts
export const authorize = <I, O>(
    operation: Operation<I, O>,
    authorization: Authorization
): AuthorizedOperation<I, O>
```

> The source code for this function is omitted for brevity. You can check it out [here](https://github.com/BanklessDAO/content-gateway/blob/develop/libs/shared/util-auth/src/Authorization.ts#L57)

then can take an `Operation` and this `Authorization` object and it will produce an `AuthorizedOperation` that can be called.

What's important to note here is that once you have an `AuthorizedOperation` you can call it wherever you want, there are no additional steps that you have to make.

### Authorizing our Operations

Now let's see an actual working example! We're going to write a very simple `Todo` app that allows listing, viewing, completing and deleting `Todo`s. The `Todo` looks like this:

```ts
import * as O from "fp-ts/Option";
import { Entity } from "../Entity";

export interface Todo extends Entity<number> {
    id: number;
    description: O.Option<string>;
    completed: O.Option<boolean>;
    published: O.Option<boolean>;
}
```

> Note that `Option` here is a wrapper object (just like Either). It represents a value (called `some`) or nothing (called `none`). We'll see why this is important in a bit.

We haven't seen `Entity` before, it is just an object that has an `owner`:

```ts
export interface Entity<ID extends number | string> {
    owner: User<ID>;
}
```

In our application we're going to have 3 different roles:

```ts
export const roles = {
    anonymous: "anonymous",
    user: "user",
    admin: "admin",
} as const;
```

These are the names we'll use when we defien the `Role`s. Before we do that let's see how the *operations* look like:

```ts
import * as O from "fp-ts/Option";
import * as TE from "fp-ts/TaskEither";
import { TodoNotFoundError } from "./errors";
import { todos } from "./fixtures";
import { Todo } from "./Todo";

export const findAllTodos = () => {
    return TE.right(Object.values(todos));
};

export const findTodo = (id: number) => {
    const todo = todos[id];
    if (todo) {
        return TE.right(todo);
    } else {
        return TE.left(new TodoNotFoundError(id));
    }
};

export const completeTodo = (input: Todo) => {
    input.completed = O.some(true);
    return TE.right(input);
};

export const deleteTodo = (input: Todo) => {
    input.completed = O.some(true);
    return TE.right(undefined);
};
```

`todos` is just an object that holds our `Todo`s mapped to their `id`s:

```ts
import * as O from "fp-ts/Option";
import { Todo } from "./Todo";

type TodoMap = {
    [key: number]: Todo;
};

export const todos: TodoMap = {
    1: {
        id: 1,
        owner: userJohn,
        description: O.some("Learn TypeScript"),
        completed: O.some(true),
        published: O.some(true),
    },
    2: {
        id: 2,
        owner: userJane,
        description: O.some("Learn fp-ts"),
        completed: O.some(false),
        published: O.some(false),
    },
    3: {
        id: 3,
        owner: adminBob,
        description: O.some("Create a typeclass"),
        completed: O.some(false),
        published: O.some(true),
    },
    4: {
        id: 4,
        owner: userJohn,
        description: O.some("Go to sleep"),
        completed: O.some(true),
        published: O.some(false),
    },
};
```

We're also going to need some `User` objects to have a complete example:

```ts
import { User } from "../User";
import { roles } from "./roles";

export const anonUser: User<number> = {
    id: 1,
    name: "anonymous",
    roles: [roles.anonymous],
};

export const userJohn: User<number> = {
    id: 2,
    name: "John Doe",
    roles: [roles.user],
};

export const userJane: User<number> = {
    id: 3,
    name: "Jane Doe",
    roles: [roles.user],
};

export const adminBob: User<number> = {
    id: 4,
    name: "Bob Doe",
    roles: [roles.admin],
};
```

What's important to not here is that we also have an `anonUser`. This object is used when somebody tries to execute operations who is *not authenticated yet*. We could have used `if`s for this, but it is just simpler to make it explicit and have a separate `User` object that explicitly contains what the anonymous user can do.

Now we're ready to define the `Authorization` object:

```ts
export const authorization: Authorization = {
    roles: {
        [roles.anonymous]: {
            name: roles.anonymous,
            permissions: anonymousPermissions,
        },
        [roles.user]: {
            name: roles.user,
            permissions: userPermissions,
        },
        [roles.admin]: {
            name: roles.admin,
            permissions: adminPermissions,
        },
    },
};
```

The actual *permissions* are just a list of `Permission` objects:

```ts
const anonymousPermissions: AnyPermission[] = [
    allowFindPublishedTodosForAnon,
    allowFindTodoForAnybody,
];

const userPermissions: AnyPermission[] = [
    allowFindPublishedTodosForUser,
    allowFindTodoForAnybody,
    allowCompleteTodoForSelf,
    allowDeleteTodoForSelf,
];

const adminPermissions: AnyPermission[] = [
    allowFindTodosForAdmin,
    allowFindTodoForAnybody,
    allowCompleteTodoForSelf,
    allowDeleteTodoForAll,
];
```

They define what *operations* can be executed for a given role and *how*:

```ts
const allowFindPublishedTodosForAnon: Permission<void, Todo[]> = {
    name: "Allow find all todos for anybody",
    operation: findAllTodos,
    policies: [allowAllPolicy()],
    filters: [filterOnlyPublished(), filterCompletedVisibilityForAnon()],
};

const allowFindPublishedTodosForUser: Permission<void, Todo[]> = {
    name: "Allow find all todos for user",
    operation: findAllTodos,
    policies: [allowAllPolicy()],
    filters: [filterOnlyPublished()],
};

const allowFindTodosForAdmin: Permission<void, Todo[]> = {
    name: "Allow find all todos for user",
    operation: findAllTodos,
    policies: [allowAllPolicy()],
};

const allowFindTodoForAnybody: Permission<number, Todo> = {
    name: "Allow find todo for anybody",
    operation: findTodo,
    policies: [allowAllPolicy()],
};

const allowCompleteTodoForSelf: Permission<Todo, Todo> = {
    name: "Allow complete todo for self",
    operation: completeTodo,
    policies: [allowForSelfPolicy()],
};

const allowDeleteTodoForSelf: Permission<Todo, void> = {
    name: "Allow delete todo for self",
    operation: deleteTodo,
    policies: [allowForSelfPolicy()],
};

const allowDeleteTodoForAll: Permission<Todo, void> = {
    name: "Allow delete todo for all",
    operation: deleteTodo,
    policies: [allowAllPolicy()],
};
```

This looks rather straightforward, but we haven't taken a look at how *policies* and *filters* can be implemented. We don't have many, but there are some very interesting use cases.

`allowAllPolicy` might be the simplest one:

```ts
const allowAllPolicy = <I> (context: Context<I>) => TE.right(context);
```

It is just a pass-through, it will allow the operation for everybody.

`allowForSelf` is a bit more elaborate:

```ts
const allowForSelfPolicy = (context: Context<I>) => {
        const { user, data } = context;
        if (user.id === data.owner.id) {
            return TE.right(context);
        } else {
            return TE.left(new MissingPermissionError());
        }
    };
```

What it does is that it checks the `owner` of the `data` against the `user` that's trying to execute the operation and will only allow it if they are the same.

`filterOnlyPublished` is a `Filter`:

```ts
const filterOnlyPublished = () => (context: Context<Todo[]>) => {
    const { data } = context;
    return TE.right({
        ...context,
        data: data.filter((d) => {
            return pipe(
                O.sequenceArray([d.published, O.some(true)]),
                O.map(([a, b]) => a === b),
                O.fold(
                    () => false,
                    (x) => x
                )
            );
        }),
    });
};
```

> The `pipe` function calls the functions in sequence and feeds the result of the previous one into the next. It is the same as if we were calling `O.fold(O.map(O.sequenceArray(...)))`.

It will take a look at the *output* of the `Operation` and will filter out all entries that aren't `published` yet.

What's interesting to note here is that the *fields* of the `Todo` are "lifted" into an `Option`. What this means is that they are not plain primitive values, but they are wrapped in a *context*. `Option` is a context that repesents the possibility of a value being `None`.

In order to work with this effectively we need to use the sequencing operations of *fp-ts* (`sequenceArray` in this case).
What it does is that it *unwraps* all the values from the `Option`s and presents them in a single array.

Similar to how `map` works in plain old javascript, the callback we give to `O.map` will only be called if there are values to check (The `Option` is not a `none`).

We use `fold` to *unwrap* the result of the mapping. If it is a `none` we'll return `false`, otherwise we'll return the value.

Now let's see why we needed these `Option`s in the first place! It is for filtering within the `Todo` itself:

```ts
const filterCompletedVisibilityForAnon = () => (context: Context<Todo[]>) => {
    const { data } = context;
    return TE.right({
        ...context,
        data: data.map((d) => {
            return {
                id: d.id,
                owner: d.owner,
                description: d.description,
                completed: O.none,
                published: d.published,
            };
        }),
    });
};
```

What `filterCompletedVisibilityForAnon` does is maps all the `Todo`s and sets the `completed` field to `none` (hides it).

> Why didn't we use simple `null`s or `undefined`? The reason is that it is often hard to tell (or impossible) why a field's value is `null` or `undefined`. `none` on the other hand explicitly states that the field is `none`. It wasn't improperly loaded from the database for example. With this pattern we can avoid a whole category of possible bug sources.


## Putting it All Together

Now we have everything in place to execute actual operations that are authorized! Let's see how this works in practice!

### Find All

For this we'll authorize the `findAll` operation:

```ts
const authorizedFindAll = authorize(findAllTodos, authorization);

const anonContext: Context<number> = {
    user: anonUser,
    data: 1,
};

const result = authorizedFindAll(
    TE.right({ user: anonUser, data: undefined })
)
```

Here `result` will be:

```json
[
    {
        "id": 1,
        "owner": {
            "id": 2,
            "name": "John Doe",
            "roles": ["user"]
        },
        "description": {
            "_tag": "Some",
            "value": "Learn TypeScript"
        },
        "completed": {
            "_tag": "None"
        },
        "published": {
            "_tag": "Some",
            "value": true
        }
    },
    {
        "id": 3,
        "owner": {
            "id": 4,
            "name": "Bob Doe",
            "roles": ["admin"]
        },
        "description": {
            "_tag": "Some",
            "value": "Create a typeclass"
        },
        "completed": {
            "_tag": "None"
        },
        "published": {
            "_tag": "Some",
            "value": true
        }
    }
]
```

Note that the `completed` field is `none` because the `user` is anonymous.


### Unauthorized Deletion

Now let's try to perform a delete as an *anon* user:

```ts
const authorizedFind = authorize(findTodo, authorization);
const authorizedDelete = authorize(deleteTodo, authorization);

pipe(
    TE.right(anonContext),
    authorizedFind,
    authorizedDelete
)
```

The result will be an `AuthorizationError` because the `user` is anonymous.

Note that these operations line up nicely because of the shape of the `AuthorizedOperation`. The result of `authorizedFind` can be fed into `authorizedDelete`. We can kick off the `pipe` with a `TE.right` of the `anonContext`.

If we used another context object:

```ts
const janesContext: Context<number> = {
    user: userJane,
    data: 2,
};
```

then the result would have been `undefined` (success) becuase the `Todo` with the id `2` is owned by *Jane*.

## Conclusion

Congratulations! You've learned how to create a simple authorization system for your application. Feel free to leave comments below if you have any questions or want to share your own ideas. 

Now let's, go forth and **kode on**!