# Tinioc

[![Node.js CI](https://github.com/tlaanemaa/tinioc/actions/workflows/node.js.yml/badge.svg?branch=main)](https://github.com/tlaanemaa/tinioc/actions/workflows/node.js.yml)

_A tiny inversion of control container for all coding styles_

## Overview

Tinioc gives you the main benefits of inversion of control (IoC) in a simple, minimal package. This means that you'll get:

- Decoupling
- Ease of testing
- A simple IoC container that's easy to understand
- Almost no constraints on your coding style

Inversion of control (IoC) brings massive benefits but applying it often means using a library that does things under the hood which might not be obvious, some magic is happening. This tends to drive people away because, as engineers, we like to know how our stuff works. That is compounded by the fact that IoC libraries often constraint you to some specific coding style, most often object-oriented design with classes.

Tinioc solves that. The library's dead simple, the whole container implementation is around 100 lines of simple code so you can easily go through it in one sitting. It also sets almost no constraints on your coding style, you can use functions or classes or whatever you like to use. The only constraint placed is that your components should be registered as factory functions, within that constraint you're free to do whatever you want. This simplicity and freedom are enabled by the simple concept of an injector function. Tinioc doesn't build dependency graphs, it doesn't even deal with dependency scopes, all it does is give you an injector function to inject your dependencies where you need them. This way you're free to use it however you want.

Here's an example of how the injector function is used to inject a dependency into a component:

```ts
// myComponent.ts

import { Inject } from "tinioc";
import { INumbersDB, NUMBERS_DB } from "../bindings";

export const myComponent = (inject: Inject): IMyComponent => ({
  getMyFavoriteNumber: async () => {
    const numbersDB = inject<INumbersDB>(NUMBERS_DB);
    const favoriteNumber = await numbersDB.getById("favorite_number");
    return favoriteNumber ?? 7;
  },
});
```

As you can see, injecting a dependency is as easy as calling a function, and it's also type-safe if you're using typescript!
You may notice that we use an id and a type separately to inject the logger here, this is how we get the IoC benefits. The concrete logger implementation isn't mentioned anywhere so we can change the implementation and as long as it fits within the same interface, we're good! Our dependant components are happy because they get a dependency whose interface they can trust and our dependency is happy because it's free to change within that interface. Decoupling!

There are also testing benefits here, we can easily pass in a mocked inject function that returns mocked dependencies.

### Dependency injections

As mentioned above, dependency injections are done with the `inject` function, provided to your component's factory as the first argument. This function can also be passed on if you find a need for that, for example into a class constructor, up to you!  
The function doesn't guarantee any type information on its own. This seems like a downside at first but is what enables us to perform IoC well. You see, we don't want to touch the implementation in the injection process, we just want the ID so that we can keep them decoupled. The type <-> id pair will be kept in a bindings file, close to each other, so it's easy to find and use.

### Bindings

An Id <-> type pair looks something like this:

```ts
// bindings.ts

export const NUMBERS_DB = Symbol.for("numbers_db");
export interface INumbersDB {
  getById(id: string): number | undefined;
  setById(id: string, value: number): void;
}

export const MY_COMPONENT = Symbol.for("my_component");
export interface IMyComponent {
  getMyFavoriteNumber(): number;
}
```

This is what we'll be using to inject our dependency and this is also what we'll be writing our implementation against. Keeping the bindings in a central place like this gives us a simple overview of all the components we've got in our system.

An easier (but not ideal) way to create id <-> type pairs is to keep them right next to your implementation. This seems convenient, its all in one place and you can derive the interface from the implementation, less work. The problem is, it doesn't actually give much decoupling. That's because you're effectively depending on the implementation and not an abstraction, the interface. You also wont get the same typings help, if you introduce a breaking change to your component you wont be notified until you look at the components you broke.  
Nevertheless, here's an example of that:

```ts
// myComponent.ts

import { Inject } from "tinioc";
import { INumbersDB, NUMBERS_DB } from "../bindings";

export const myComponent = (inject: Inject) => ({
  getMyFavoriteNumber: async () => {
    const numbersDB = inject<INumbersDB>(NUMBERS_DB);
    const favoriteNumber = await numbersDB.getById("favorite_number");
    return favoriteNumber ?? 7;
  },
});

export const MY_COMPONENT = Symbol.for("my_component");
export type IMyComponent = ReturnType<typeof myComponent>;
```

### Container

To facilitate the injection, we need a dependency injection container. This is also where our interfaces, ids, and implementations will be connected. Here's an example of a container:

```ts
// container.ts

import { Container } from "tinioc";
import * as bindings from "./bindings";
import { numbersDB } from "./numbersDB";
import { myComponent } from "./database/numbersDB";

export const container = new Container();

container.bind<bindings.INumbersDB>(bindings.NUMBERS_DB, numbersDB);
container.bind<bindings.IMyComponent>(bindings.MY_COMPONENT, myComponent);
```

That's it! Now you've got a fully functional IoC container set up.  
For smaller applications, it's usually enough to use a single global container but you can also split your project into submodules with each having its own container. Then, if one submodule needs to access dependencies from another, you'll just use `container.extend` to create a parent-child relationship between them.

### Getting access to the components

Getting access to components is very straightforward, here's an example of an express controller using a component:

```ts
// controller.ts

import { RequestHandler } from "express";
import { IMyComponent, MY_COMPONENT } from "./bindings";
import { container } from "./container";

export const controller: RequestHandler = async (req, res, next) => {
  const myFavoriteNumber = await container
    .get<IMyComponent>(MY_COMPONENT)
    .getMyFavoriteNumber();

  res.json({ myFavoriteNumber });
};
```

Notice how we use the `get` method in the same way as we used `inject` before, that's because they're the same method!  
Now, let's say you want to put some request context into the container before you get your component. This can be easily achieved with `container.createChild` like so:

```ts
// controller.ts

import { RequestHandler } from "express";
import { IMyComponent, MY_COMPONENT } from "./bindings";
import { container } from "./container";

export const controller: RequestHandler = async (req, res, next) => {
  const ctx = { correlationId: req.header("correlation-id") };
  const myFavoriteNumber = await container
    .createChild()
    .bind<IRequestContext>(REQUEST_CONTEXT, () => ctx)
    .get<IMyComponent>(MY_COMPONENT)
    .getMyFavoriteNumber();

  res.json({ myFavoriteNumber });
};
```

And just like that, you've got your request context available everywhere within that child container. Just be sure to create a child container when adding transient components like that tho, else you'll pollute your root container. The child container makes use of the same child-parent relationship we saw earlier with `container.extend`, it just does it the other way around.

If for some reason the component is not found, a `BindingNotFoundError` will be thrown. You can also import that error class from `tinioc` and use it in `instanceof` checks to handle that specific error if you wish.

## Container API

### container.bind()

Register an id with a component in the container.

You can make use of the generic type on this method to enforce that the
registered component matches the required interface.

Example:

```ts
container.bind<IMyComponent>(MY_COMPONENT, myComponent);
```

The registered binding can later be injected with the `inject` function like so:

```ts
const component = inject<IMyComponent>(MY_COMPONENT);
```

It is suggested you keep your dependency ids and types close to each other,
preferably in a separate `bindings` file. That makes them easy to use and improves
maintainability.

### container.isBound()

Check if there is a binding for a given id.
This will check this container and also all of it's parents.

Example:

```ts
const myComponentIsBound = container.isBound(MY_COMPONENT);
```

### container.isCurrentBound()

Check if there is a binding for a given id.
This will check only this container.

Example:

```ts
const myComponentIsBound = container.isCurrentBound(MY_COMPONENT);
```

### container.unbind()

Removes the binding for the given id.
This will only remove it in this container.

Example:

```ts
container.unbind(MY_COMPONENT);
```

### container.get()

Get a binding from the container

The binding will be first looked for in this container.
If it's not found here, it will be looked for in parents, in their order in the `parents` array.

- If the binding is found then its initialized and returned.
- If it's not found then a `BindingNotFoundError`
  is thrown

Example:

```ts
const component = container.get<IMyComponent>(MY_COMPONENT);
```

### container.createChild()

Creates and returns a child container.

This is effectively the reverse of extending.
The new container will have this container as the only parent.

Child containers are very useful when you want to bind something for a single run,
for example, if you've got request context you want to bind to the container before getting your service.
Using child containers allows you to bind these temporary values without polluting the root container.

Example:

```ts
const child = container.createChild();
```

### container.extend()

Extends the container's array of parents with the given containers.
This makes the given containers' contents available to this container,
effectively creating a parent-child relationship.

For example, if some components in your container depend on some components
in another container, then you should extend your container with that other container,
to make those dependencies available for your components.

This will append to the list of parents and not overwrite it.
A new parent is only added if it doesn't already exist in the `parents` array.

Example:

```ts
container.extend(otherContainer1, otherContainer2);
```

### container.parents

Array of parent containers

These allow setting up parent-child relationships between containers, thus enabling
hierarchical dependency injection systems. Multiple parents are supported, so you can essentially
make your container "inherit" from several other containers

### All type declarations

```ts
declare type ID = string | symbol;
declare type Inject = <T>(id: ID) => T;
declare type FactoryOf<T> = (inject: Inject) => T;

declare class Container {
  parents: Container[];
  bind<T>(id: ID, value: FactoryOf<T>): this;
  isBound(id: ID): boolean;
  isCurrentBound(id: ID): boolean;
  unbind(id: ID): this;
  get<T>(id: ID): T;
  extend(...containers: Container[]): this;
  createChild(): Container;
}

declare class BindingNotFoundError extends Error {
  constructor(id: string | symbol);
}
```

## Example

I know engineers like to tinker with stuff so I've created a fully functional microservice that showcases how tinioc is used. You can find it here: https://github.com/tlaanemaa/tinioc-example
