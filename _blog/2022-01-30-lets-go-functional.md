---
excerpt: "Functional programming is a really hot topic nowadays but is it really that useful? In this article we're going to explore this topic."
title: "Let's go functional?"
tags: [functional-programming, typescript]
author: addamsson
short_title: "Let's go functional?"
series: content-gateway
comments: true
updated_at: 2022-01-30
---

> In this post we're going to take a look at why we chose FP tools (like fp-ts and io-ts) for *Content Gateway* and why its pros outweighs the cons.

I've been thinking a lot about functional programming in the last few years and I've also contributed to some FP libraries as well. My main goal with all of this was to determine whether the hype around this topic is warranted or not. After working on programs using the functional paradigm in the last few years I concluded that the answer to that questions is the usual "It depends".

There are people out there who will try to sell FP as a panacea and a solution for all your problems, but I think that - as with many other things - you'll have to find the right balance between FP and OOP (or whatever else) for your own problem. I've seen projects go **all in** on FP just to go down in flames after the only contributor that really understood what a `ReaderTaskEither` was left the project. I've also seen places where they were using just OOP and the codebase was full of brittle inheritance structures that would make your eyes water if you took a look at them.

I think a better way is to be open and pick the right tool for the job. In other words pragmatism beats purism when you want to ship actual products. Therefore the last thing I want to see on anybody's resume who wants to join the project I'm working on is a sentence like "I'm a X programmer". Some people even go so far as to say that they are *ember.js* programmers. Why? Just to cite a classic:

> The more labels you have for yourself, the dumber they make you.

So without further ado, let's see why we chose to ~~go functional~~ use some FP tools on the project.


## The Elephant in the Room

Most of you probably know that Javascript has *some* design flaws that mostly come from the very simplistic design choices that were made when the language was first created. I'm not here to bash Javascript, but it is a fact that it carries some *baggage* from the olden days.

What's more problematic is that unlike Python for example Javascript doesn't have clear coding standards or best practices. In this sense it is very much like Scala where you can put two JS (or Scala) devs on a project and they probably won't agree on how to do `x`. There are problems that you face every day and having some framework (not necessarily expressed in code, but more like a set of best practices) goes a long way in helping you have a consistent codebase.

Maybe this is the reason why many projects choose to use an *actual framework*. While this looks like a good idea on the surface, the problem with them is that they mostly use arbitrary coding standards. If you learn how to use framework `x`, you probably can't apply it to framework `y`... in fact there is a fairly good chance that a best practice in `x` is an antipattern in `y`. 🤷‍♂️ Another issue is that by choosing `x` you really have to understand the thought process of some developer who dreamed up `x` in the first place. Are they really that good at what they do? Can they design something that will work for their users in the long run?

This is where FP comes into the picture. It has its roots in mathematics, and no matter what language you use you'll see the same constructs (and mostly the same names). What's even better is that there is usually only one way to do something and if you try to do it some other way you'll end up with *horrible terrible* code. The literature on FP is also very good, if you really want to learn it you *can do it*.

The technique itself is a bit restrictive, but not too restrictive, you can still apply it to almost anything.

> 📗 One example where it didn't work for me is recursive data processing. I know that there are FP constructs to do that but my library of choice didn't have support. 😒

> 📙 If you ever encounter something that you can't solve with FP (or it is really hard to do) then just don't do it. As [Rich Hickey](https://changelog.com/posts/rich-hickeys-greatest-hits) said once: no one cares about what happens **in** your function as long as it doesn't break the outside world and it adheres to its contract (input / output). What this means in practice is that if you think that expressing something would be an excruciating task, but you still have to return that `Either` than just wrap the whole thing in a `tryCatch` and write it in a way that you see fit.

In FP there are two basic constructs: *functions* and *wrappers*. Functions express your domain logic and wrappers express cross-cutting concerns. 

> 📗 I intentionally used the word "wrapper" here. I'm not a big fan of using fancy technical terms when something simpler would get the message across for more people.

These functions are usually chained together to form a composite operation. What this usually means is that the result of a function is *piped into* the next function:

```ts
second(first(x))
```

This pattern is so common that there is a function that makes this much more readable:

```ts
pipe(first(x), second)
```

There, you have it! FP in a nutshell.

> I'm just kidding. 👀

The function parts and the piping is probably easy to understand but what the heck did I mean by "wrappers"? Let's take a look at some examples where using FP is probably a good choice, to explain them!


## FP: the Good Parts

### Error Handling

One of the major problems in Javascript (and as an extension in Typescript) is that whenever you use `try`/`catch` the error you get is of type `unknown`. This means that you'll have to go through the painful process of trying to figure out what kind of error you received.

Another problem with error handling in JS is that it breaks the control flow.

> 📗 Fun fact: I knew an architect who used this "feature" of exceptions to implement *goto* in Java. I still thank whatever higher power there is every day for not making me work on that codebase!

So what can we do to fix this? Luckily for us, our FP library of choice (fp-ts) has a way to deal with this problem with a construct called `Either`.

> 📗 Even though learning all these arcane names might be daunting at first, at some point you'll realize that you only have to learn it **once**. If you choose to go and learn [Kotlin](https://kotlinlang.org/) for example and start using [Arrow](https://arrow-kt.io/) you'll see that they have the (more or less) same nomenclature! The reason for this is that they are both rooted in the same mathematical constructs.

`Either` is one of those *wrappers* I mentioned that are there to solve cross-cutting concerns. This concern in our case is error handling. We already know that FP is all about creating pipelines that are compositions of functions expressing our domain logic, with some wrappers sprinkled on top. `Either` lets us capture error states in our program. Let's see how it works. This is how a classic `try`/`catch` looks like in Typescript:

```ts
export const someOperation = () => {
    try {
        throw Error("x");
    } catch (e: unknown) {
        console.error("Oh no.", e);
    }
};
```

Here we capture all error states in `catch`. We can also see the type problem (`e` is of type `unknown`) that we're trying to address. So what happens if we just rethrow?

```ts
export const someOperation = () => {
    try {
        throw Error("x");
    } catch (e: unknown) {
        console.error("Oh no.", e);
        throw new Error("Can't perform some operation.");
    }
};
```

What will happen is that whoever calls `someOperation` will have a chance to get an `Error` and we have no way of signaling that this is a possibility! Let's see how `Either` helps us:

```ts
// 👇 it is common to import Either like this so that we can use all operations it provides
import * as E from "fp-ts/Either";

export const someOperation = (): E.Either<Error, void> => {
    return E.tryCatch(
        () => {
            throw Error("x");
        },
        (e: unknown) => {
            console.error("Oops!", e);
            return new Error("Can't perform some operation.");
        }
    );
};
```

As you can see, instead of not returning anything (implicitly returning `void`) now we return `Either<Error, void>`. This is why I called this a *wrapper*. It wraps our `void` and adds some extra functionality on top. In our case it encodes the possible error states of a synchronous operation. There is also a semantic purpose of `Either`. If you follow the best practices and your users are also familiar with what an `Either` is you can be sure that a function that returns an `Either` will never *throw*!


### A Note on Purity

If you've been using FP for a while you might say that `Oh no! You did a side effect! Heresy!`. I think this is a common mistake one can make when using FP. The "side effect" here is the `console.error` statement. It performs an IO operation (writes to the console). In FP IO (or any other side effect) shouldn't be performed like this.

I think this is nonsense and this is the line I draw in the sand when I try to balance efficiency and purity. I trust myself and I also trust the developers I work with to *know* what can be safely done and what can't. We have code reviews for a reason.

So from now on you'll see some *heresies* like this, but know that the reason I'm not doing it the "pure" way is that I think the cons of going "all the way" greatly outweigh the benefits. After all this article is about my personal opinions on how to work effectively with FP constructs, and not a scholarly article.

Now we have an idea why these wrappers are useful and why they help with establishing useful semantic patterns within our application.

> 📗 Since this article is a hands-on introduction I'm not going into detail with regards to typeclasses, monads, and all the good stuff you'll probably hear about elsewhere. If you're interested I'll share some links at the end! 👀

Now let's move on to our next topic... 👀


### Error Handling the Right Way

We've seen how to use an `Either` instead of a `try`/`catch` to handle error states. But what if we don't have a `try`/`catch`, but the error state comes from something else? A good example is validation. Let's see how to create `Either`s by hand by writing a small validator function:

```ts
export const validateLength = (
    value: string,
    min: number,
    max: number
): string => {
    if (value.length < min) {
        throw new Error(`Must be at least ${min} characters long`);
    }
    if (value.length > max) {
        throw new Error(`Must be at most ${max} characters long`);
    }
    return value;
};
```

This function takes a `string` and some constraints and returns the `string` unless it is invalid. In that case we get an `Error`. We've already discussed why the built-in error handling is problematic, so let's see how we can use `Either` to solve this problem.

First of all we're going to need to modify the signature:

```ts
import * as E from "fp-ts/Either";

export const validateLength = (
    value: string,
    min: number,
    max: number
): E.Either<Error, string> => {
    if (value.length < min) {
        throw new Error(`Must be at least ${min} characters long`);
    }
    if (value.length > max) {
        throw new Error(`Must be at most ${max} characters long`);
    }
    return value;
};
```

`Either` has some *constructors* that you can use to create instances of it. The error state is created by `left`, and the happy case is created by `right`. We're also going to lose the `throw` as we'll wrap the errors instead:

```ts
import * as E from "fp-ts/Either";

export const validateLength = (
    value: string,
    min: number,
    max: number
): E.Either<Error, string> => {
    if (value.length < min) {
        return E.left(new Error(`Must be at least ${min} characters long`));
    }
    if (value.length > max) {
        return E.left(new Error(`Must be at most ${max} characters long`));
    }
    return E.right(value);
};
```

There you go. This operation will never `throw`, and the semantics are also clear. Now you see why we imported `Either` with the `import * as E` syntax: because we need not just the type, but the constructors too.

We've covered **most** operations that you will use in your day-to-day. There is more, but we'll cover them in the next article.


### Error Types

If you've been paying attention you might be wondering now: "Ok, I get it, but we still have `Error`s. How is this better than an `unknown`? I still have to do the type checking!". There is a solution for that: **discriminated unions**. Explaining the concept is out of scope for this article but the gist of it is that if you have a field in a type that will have an unique value for each subtype of that type, then you can use it to discriminate between the subtypes. This field is usually called a `_tag`. So for example:

```ts
export interface ProgramError {
    _tag: string;
    message: string;
}
```

this can serve as a good base for a discriminated union. Now we can have a base class for all our errors for example:

```ts
export abstract class ProgramErrorBase<T extends string>
    extends Error
    implements ProgramError
{
    public _tag: T;
    public message: string;

    constructor(params: {
        _tag: T;
        message: string;
    }) {
        super(params.message);
        this._tag = params._tag;
        this.message = params.message;
    }
}
```

which makes creating `Error`s a breeze:

```ts
export class SchemaValidationError extends ProgramErrorBase<"SchemaValidationError"> {
    constructor() {
        super({
            _tag: "SchemaValidationError",
            message: `Schema validation failed`
        });
    }
}
```

With this we can put the concrete error type(s) in our `Either`s:

```ts
import * as E from "fp-ts/Either";

export const validateLength = (
    value: string,
    min: number,
    max: number
): E.Either<StringTooShortError | StringTooLongError, string> => {
    if (value.length < min) {
        return E.left(new StringTooShortError(`Must be at least ${min} characters long`));
    }
    if (value.length > max) {
        return E.left(new StringTooLongError(`Must be at most ${max} characters long`));
    }
    return E.right(value);
};
```

that can be checked exhaustively:


```ts
export const someOperation = () => {
    const result = validateLength("hey", 1, 10);
    if (E.isLeft(result)) {
        let msg: string;
        const tag = result.left._tag;
        switch(tag) {
            case "StringTooShortError":
                msg = "String too short";
                break;
            case "StringTooLongError":
                msg = "String too long";
                break;
        }
        console.log(msg);
    } else {
        console.log("success");
    }
}
```

If you remove the `case "StringTooLongError":` branch you'll get a compile error: `Variable 'msg' is used before being assigned`. 💪

So what if you don't use things like `msg`? How can you be sure that your check is exhaustive? That's an *absurd* question. Literally:

```ts
const tag = result.left._tag;
switch (tag) {
    case "StringTooShortError":
        console.log("String too short.");
        break;
}
```

👆 This will compile, but we can use a function that will make it fail:

```ts
import { absurd } from "fp-ts/lib/function";

const tag = result.left._tag;
switch (tag) {
    case "StringTooShortError":
        console.log("String too short.");
        break;
    default:
        absurd(tag);
        // 👆 Argument of type 'string' is not assignable to parameter of type 'never'
}
```

The only way to make this compile is to be exhaustive:

```ts
const tag = result.left._tag;
switch (tag) {
    case "StringTooShortError":
        console.log("String too short.");
        break;
    case "StringTooLongError":
        console.log("String too long.");
        break;
    default:
        absurd(tag);
}
```

Not so absurd after all! 👀


## Asynchronous Programming

`Either` is only one of many components and if you want to have a more complete toolbox you'll need to dig a bit deeper. Let's think about something else that's also notoriously difficult to use properly: async/await (aka: `Promise`s). As [James Snell outlines in his talk](https://www.youtube.com/watch?v=XV-u_Ow47s0&ab_channel=node.js), if you have performance problems in Javascript it is highly likely that they have something to do with `Promise`s.

So what is a `Promise`? It is a construct that represents a value that will be available at some point in the future.

> 📗 It sounds like a cross-cutting concern that can be expressed by a wrapper isn' it?

 Before `Promises` the way we represented this was to use callbacks and [Trampolines](https://en.wikipedia.org/wiki/Trampoline_(computing)). Javascript is still a language that's single-threaded, so

> 📗 I know there are workers, but they are nowhere as useful as `Thread`s in Java for example.

we need a way to do away with all the complexity of these old constructs (and also the callback hell). The problem is that it is very easy to go wrong. When do I use the `async` keyword? What happens if I throw in an `async` function? When do I call `Promise.resolve`? When do I `reject`? What happens if I do so? Questions, questions. What's also not helpful is that we can mix and match the `then` + `catch` and the `async` + `await` syntax. We're facing the same kind of problem as with error handling: there is no "best practice" to follow or agreed-on semantics that everybody uses the same way.

Thankfully we have some *wrappers* that help with this. Enter the `Task`. If we take a look at the definition of a `Task`:

```ts
export interface Task<A> {
  (): Promise<A>
}
```

it is just a function that returns a `Promise`. This is how you can create a task:

```ts
import * as T from "fp-ts/Task";

const loadData = (): T.Task<string> => {
    return T.of("data");
}
```

So...why would you do that? Again, it is semantics. The above code is technically equivalent to doing this:

```ts
const loadData = (): Promise<string> => {
    return Promise.resolve("data");
}
```

But whenever you see a `Task` you can *expect* a few things.

- A `Task` represents an asynchronous operation.
- A `Task` **will never fail**.

So if something returns one you won't have to care about error handling. For example the `parseInt` is a function that will never fail. The worst case is that you'll get a `NaN`.

So what if I have an operation that might fail, but I still want to represent it with a `Task`? Well, you can use a `try`/`catch` block and return a result based on the outcome:

```ts
import * as T from "fp-ts/Task";

const loadData = (): T.Task<string> => {
    try {
        return T.of("data");
    } catch (e) {
        return T.of("error");
    }
}
```

## Asynchronous Operations That Can Fail

You might already say "Wait a second, isn't this something that can be done with `Either`?" and you are right. We have a *wrapper* that combines an asynchronous operation with error handling, the `TaskEither`. It is a construct that represents an *asynchronous operation* that can fail. Let's say that you have something like this:

```ts
export const loadData = async () => {
    const result = await fetch("/api/data");
    if (result.status === 200) {
        return result.json();
    } else {
        throw new Error("Failed to load data");
    }
};
```

There are a couple of problems here. First, the `fetch` call might fail. We don't handle that here. Second we also throw our own `Error`s in case the `status` was `200`. This is a mess. How can we improve on this? `TaskEither` to the rescue!

```ts
import * as TE from "fp-ts/TaskEither";
import { pipe } from "fp-ts/lib/function";

export const loadData = (): TE.TaskEither<Error, string> => {
    return pipe(
        TE.tryCatch(
            () => fetch("/api/data"),
            (e) => new Error(`Failed to load data: ${e}`)
        ),
        TE.chain((result) => {
            if (result.status === 200) {
                return TE.tryCatch(
                    () => result.json(),
                    (e) => new Error(`Json conversion failed: ${e}`)
                );
            } else {
                return TE.left(new Error("Failed to load data"));
            }
        }),
    );
}
```

Wait a second! What's going on here? As it turns out there was another case that we didn't handle in the previous implementation: the possible error states of the `result.json()` call! This is something that happens very often with Javascript and sometimes with Typescript when we use type inference. We thought that we were returning a `string`, but in fact we were returning a `Promise<string>`!

With `TaskEither` we can't accidentally return the wrong type because the compiler will scream at us much earlier.

Also...what's this `pipe` thing? Let's unwrap what we see in the code above:

```ts
TE.tryCatch(
    () => fetch("/api/data"),
    (e) => new Error(`Failed to load data: ${e}`)
)
```

👆 This is the same as `E.tryCatch`, but it will return a `TaskEither` (asynchronous operation that can fail) instead of an `Either`.

`pipe` is a function that lets us thread functions together (the same as the [Proposed Pipe Operator](https://github.com/tc39/proposal-pipeline-operator)). What it does is that it evaluates the first parameter, and passes it to the function in the second parameter, and so on. So for example with this:

```ts
pipe(
    1,
    (x) => x + 1,
)
```

We'll get `2` as a result because the `1` was passed to the function in the second parameter.

If you decide to give this approach a try you'll use `pipe` **all the time**.

> 📗 Note that there are similar functions like `flow`, but we don't cover them here. Maybe later in another article.

So what's the deal with the `TE.chain`? The thing is that we want to return a `TaskEither`, but we need to perform some transformations on the initial one. For example what we do in the code is;

- call `/api/data`
- deal with its errors
- check if the result is `200`
- deal with it if it is not
- transform the result into a json string
- deal with its errors

The problem is that the first operation (calling `/api/data`) will return a `Promise`, and extracting the `json` does the same. We want the result of the first to be fed into the next one. In other words we want to *unwrap* the first one and *rewrap* it into another one. This is what `TE.chain` does. It unwraps the value that's wrapped with the `TaskEither` and lets us rewrap it in any way we see fit.

The equivalent of the above code in the "old way" would be:

```ts
export const loadData = (): Promise<string> => {
    return fetch("/api/data")
        .then((result) => {
            if (result.status === 200) {
                return result.json().catch((e) => {
                    throw new Error(`Json conversion failed: ${e}`);
                });
            } else {
                throw new Error("Failed to load data");
            }
        })
        .catch((e) => {
            throw new Error(`Failed to load data: ${e}`);
        });
};
```

The difference between the two is that instead of having a `Promise` that might reject we clearly express our **intent**. The amount of code we've written is more or less the same, but we no longer have the cognitive burden of having to think about `Promise` rejections within the function and we **know exactly** what can go wrong outside of the function.

> 📗 Note that we didn't use specific error types here for simplicity's sake.


## Minefields

There other wrappers that you might see in other codebases, but some of them are dangerous. The two that you might see the most is `Option` and `TaskOption`. Option represents an operation that *might* return a value:

```ts
const findUser = (id: number): O.Option<User> => {
    return O.fromNullable(users.find((u) => u.id === id));
}
```

If `users.find` was an asynchronous operation we could use `TaskOption` instead:

```ts
import * as TO from "fp-ts/TaskOption";

const findUser = (id: number): TO.TaskOption<User> => {
    return TO.fromNullable(userRepository.find(id));
}
```

The problem with `Option` is that it is isomorphic with simple nullable types:

```ts
const findUser = (id: number): User | null => {
    return users.find((u) => u.id === id);
}
```

It adds no extra information about why the value wasn't present, but it adds complexity. The same stands for `TaskOption`. Why `TaskOption` is even worse is that you can do this:

```ts
import * as TO from "fp-ts/TaskOption";

const findUser = (id: number): TO.TaskOption<User> => {
    return TO.tryCatch(userRepository.find(id));
}
```

What happens if there is an error when calling `userRepository.find`? `TO.tryCatch` will just swallow it and you'll be none the wiser.

**Ouch!**

This is why I try to stay away from using either of those.


## Pipelines

We've already seen the usage of the `pipe` function with `TaskEither` but it was a simple case: `TaskEither` went in, and another `TaskEither` came out.

There are some problems that you'll inevitably face when working with `pipe`s so we're going to take a look at some of them now.

> 📗 There are some more complex cases that we'll cover in the next article.


### Mapping

Let's say that we have an operation that rewraps an `Either` to another one:

```ts
export const loadData = (): E.Either<Error, string> => {
    return pipe(
        E.tryCatch(() => {
            return fetchData();
        }, (e) => new Error(`It didn't work: ${e}`)),
        E.chain((result) => {
            return E.right(result + "ok");
        })
    );
};
```

> 📙 `E.chain` will only be called when the previous `Either` is a `Right`. Errors short-circuit the chain so you'll never have to deal with them in a `chain` or a `map` call.

As you can see all we do in the `chain` call is to transform the `result`. In these cases we can simplify this to a `map` call:

```ts
pipe(
    E.tryCatch(() => {
        return fetchData();
    }, (e) => new Error(`It didn't work: ${e}`)),
    E.map((result) => {
        return result + "ok";
    })
)
```

`map` rewraps the `Either` for us and it takes a function that will calculate the next `Right` value for us.


### Rewrapping Errors

What happens a lot is that we need to deal with errors produced by others. For example what happens if we get an `Either` but its `Left` value is not an `Error` that has a discriminator field? We can use `mapLeft` in this case:

```ts
pipe(
    fetchData(),
    E.mapLeft((e) => new MyDiscriminatedError(`Failed to fetch data: ${e}`)),
);
```

If we take a look at what type will this produce:

```ts
const result: E.Either<MyDiscriminatedError, string> =  pipe(
    fetchData(),
    E.mapLeft((e) => new MyDiscriminatedError(`Failed to fetch data: ${e}`)),
);
```

we'll see that the `Error` is gone.


### A Note on Debugging

It will often happen that you write a nice `pipe` and it doesn't return the type that you expected. This can be extremely frustrating especially if you have a very complex `pipe`. There is something that you can do that will help you preserve your sanity. Let's take a look at some code:

```ts
pipe(
    fetchData(),
    E.mapLeft((e) => new MyDiscriminatedError(`Failed to fetch data: ${e}`)),
    E.map((result) => {
        return result + "ok";
    }),
);
```

If we assign this to a variable VS Code will show us the type of the variable so we can easily specify it and we'll be able to see what's wrong.
What also helps is to just comment the individual parameters to `pipe` and see what the types are:

Step 1:

```ts
const result: E.Either<Error, never> = pipe(
    fetchData()
    // E.mapLeft(
    //     (e) => new MyDiscriminatedError(`Failed to fetch data: ${e}`)
    // ),
    // E.map((result) => {
    //     return result + "ok";
    // })
);
```

Step 2:

```ts
const result: E.Either<MyDiscriminatedError, never> = pipe(
    fetchData(),
    E.mapLeft(
        (e) => new MyDiscriminatedError(`Failed to fetch data: ${e}`)
    ),
    // E.map((result) => {
    //     return result + "ok";
    // })
);
```

Step 3:

```ts
const result: E.Either<MyDiscriminatedError, string> = pipe(
    fetchData(),
    E.mapLeft(
        (e) => new MyDiscriminatedError(`Failed to fetch data: ${e}`)
    ),
    E.map((result) => {
        return result + "ok";
    })
);
```

👆 With this simple technique you'll be able to prevent **a lot** of the frustration that you would otherwise have to endure.


### Transforming Wrappers

The thing with `pipe` is that the output of an operation has to align with the input of the next operation. So for example if we have an `Either`, but we have to return a `TaskEither` what can we do? Let's take a look at some of the *constructors* that we can use in these cases.

Let's say that `fetchData` returns an `Option` instead of an `Either`:

```ts
const fetchData = () => O.of("data");
```

If we wanted an `Either` we can use `E.fromOption`:

```ts
export const loadData = (): E.Either<Error, string> => {
    return pipe(
        fetchData(),
        E.fromOption(() => new Error("Failed to load data")),
    );
};
```

> 📗 TE.fromTaskOption works the same way.

This will treat `None` as an error and `Some` as a `Right`.

> 📗 Some of these `from*` operations will need some additional parameters to be able to properly change the semantics of the previous wrapper. In the case of Option -> Either for example we need to supply a function that will handle the *left* case which is not represented by `Option`.

Creating a `Task` is rather simple, but how can we create a `TaskEither` out of it? It is super simple:

```ts
TE.fromTask(task);
```

"Why don't we have to supply any parameters?" you might ask. That's because `fromTask` will simply treat the rejection as an error.

`TE.fromEither` works in the same way: it will just *promisify* the `Either`, so no extra parameters are needed.


Ok, we've covered the basics, but what happens if I want to `chain` an `Either` but I also want to introduce a new error case? Let's see:

```ts
const fetchData = (): E.Either<FetchFailedError, string> => {
    return E.right("data");
};

export const loadData = (): E.Either<FetchFailedError, string> => {
    return pipe(
        fetchData(),
        E.chain((data) => {
            if (data !== "data") {
                return E.left(new DataInvalidError());
            } else {
                return E.right(data);
            }
        })
    );
};
```

I will get a rather lengthy error message babbling something about things not being assignable. Adding the new Error type to the function signature doesn't help either:

```ts
export const loadData = (): E.Either<FetchFailedError | DataInvalidError, string> => {
    // ...
};
```

So what's the problem? It is the `chain`! It expects the `Either` to preserve its shape (generic type parameters). This is the case when we need to *widen* the type. The function
that lets us do this is `chainW` (chain widen):

```ts
export const loadData = (): E.Either<FetchFailedError | DataInvalidError, string> => {
    return pipe(
        fetchData(),
        E.chainW((data) => {
            if (data !== "data") {
                return E.left(new DataInvalidError());
            } else {
                return E.right(data);
            }
        })
    );
};
```

Now the compiler error is gone.

> 📗 We'll see some more of the weird Hungarian notation later down the road. There aren't a lot of them, but unfortunately this is something we have to live with.


## Conclusion

We've learned that what FP brings to the table apart from some arcane naming conventions is a very consistent way of thinking about problems. Now we know how to mix our domain logic with *wrappers* that handle cross-cutting concerns and how to use them as buliding blocks to create pipelines. There are **much more** FP concepts that we haven't covered yet, but we'll get there.

The takeaway from all of this is that FP is just a tool that you can employ to solve some of your problems. You don't have to go **all the way** and delete all your classes to start taking advantage of what this technique brings to the table. You can just sprinkle it on your codebase where applicable and enjoy the benefits of having clear semantics and more robust code. It also lets us think about our use cases in a more fundamental way: as functions that we can compose with each other. This is a powerful technique that is often overlooked especially if someone comes from an OOP background.

In the next article we'll look at some more advanced concepts so stay tuned!

Until then, let's go forth and **kode on**!