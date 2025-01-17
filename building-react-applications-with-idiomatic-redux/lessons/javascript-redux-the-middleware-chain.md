We have written two functions that wrap the `dispatch` function to add custom behavior. Let's take a closer look at how they work together.

The final version of the `dispatch` function before we turn in the `store` is the result of calling `addPromiseSupportToDispatch`.

The function it returns acts like a normal dispatch function, but if it gets a **promise**, it waits for this promise to **resolve** and passes the result to `rawDispatch`, where `rawDispatch` is the previous value of `store.dispatch`.

For non-promises, it calls `rawDispatch` right away. `Rawdispatch` corresponds to the `store.dispatch` at the time `addpromiseSupporToDispatch` was called.

`Store.dispatch` was reassigned earlier, so it's not completely fair to refer to it as `rawDispatch`. I'm renaming `rawDispatch` to `next` because this is the next dispatch function in the chain.

**configureStore.js**
```JavaScript
const addPromiseSupportToDispatch = (store) => {
  const next = store.dispatch;
  return (action) => {
    if (typeof action.then === 'function') {
      return action.then(rawDispatch);
    }
    return rawDispatch(action);
  };
};
```
It refers to `store.dispatch` that was returned from `addLoggingToDispatch`. `AddLoggingToDispatch` also returns a function with the same API as the original dispatch function, but it logs the `action.type`, the previous state, the action, and the next state along the way.

It calls the `rawDispatch` with corresponds to `store.dispatch` at the time `addLoggingToDispatch` was called. In this case, this is the `store.dispatch` provided by create store.

However, it is entirely conceivable that we might want to override dispatch function before adding the login. For consistency, I'm going to rename `rawDispatch` to `next`, as well. It's just that in this particular case, next points to the original `store.dispatch`.

**configureStore.js**
```JavaScript
const addLoggingToDispatch = (store) => {
  const next = store.dipatch;
  if (!console.group) {
    return next;
  }
}
```
While this method of extending the store works, it's not really great at **re-override the public API** and replace it with **custom** functions. To get away from this pattern, I am declaring an array of what I'm calling `middlewares` functions, which is just a fancy name I use for these functions I wrote, and I'm pushing `addLoggingToDispatch` itself to an array, so this is an array of functions that will be applied later as a single step.

**configureStore.js**
```JavaScript
const middlewares = [];

if (process.env.NODE_ENV !== 'production') {
  middlewares.push(addLoggingToDispatch);
}

middlewares.push(addPromiseSupportToDispatch);
```
I'm also adding `addPromiseSupportToDispatch` to the `middlewares` array, and again, I don't need to apply it yet because I'll write a separate function that applies the middlewares.

I will call this function `wrapDispatchWithMiddlewares`, and I'll pass this `store` as the first argument and an array of `middlewares` as the second argument.

**configureStore.js**
```JavaScript
wrapDispatchWithMiddlewares(store, middlewares);
```
I'm adding this new function called `wrapDispatchWithMiddlewares`, and inside it I'm going to use the `middlewares` for each method. to run some code for every middleware.

Specifically, I want to override this `store.dispatch` function to point to the result of calling the middleware with the store as an argument.

**configureStore.js**
```JavaScript
const wrapDispatchWithMiddlewares = (store, middlewares) => {
  middlewares.forEach(Middleware =>
    store.dispatch = middleware(store)
  );
};
```
Inside the middleware functions themselves, there is a certain pattern that I have to repeat. I'm grabbing the value of `store.dispatch`, and I'm storing it in a variable called `next` so that I can call it later.

To make it a part of the middleware contract, I can make `next` an outside argument, just like the `store` before it and the `action` after it.

**configureStore.js**
```JavaScript
const addLoggingToDispatch = (store) => {
  return (next) => { 
    if (!console.group) {
      return next;
    }

    return (action) => { ... }
  }
}
```
This way, the middleware becomes a function that returns a function that returns a function, which is not very common in JavaScript, but is actually very common in functional programming languages, and this pattern is called [currying](https://egghead.io/lessons/javascript-what-is-currying).

Again, rather than take the next middleware from the store, I will make it intractable as an argument so that my function that calls the middlewares can choose which middleware to pass.

**configureStore.js**
```JavaScript
const addPromiseSupportToDispatch = (store) => {
  return (next) => {
    return (action) => {
      if (typeof action.then === 'function') {
        return action.then(next);
      }
      return next(action);
    };
  };
};
```
Finally, store is not the only injected argument, so I also need to inject the next middleware, which is the previous value of `store.dispatch` function.

**configureStore.js**
```JavaScript
const wrapDispatchWithMiddlewares = (store, middlewares) => {
  middlewares.forEach(Middleware =>
    store.dispatch = middleware(store)(store.dispatch)
  );
};
```
Now that middlewares are a first-class concept, I will rename `addLoggingToDispatch` to just `logger`, and I will rename `addPromiseSupporToDispatch` to the `promise` middleware.

The currying style of function declaration can get very hard to read. However, we can use [arrow functions](https://egghead.io/lessons/arrow-function) and rely on the fact that arrow functions can have expressions as their bodies.

**configureStore.js**
```javascript
const promise = (store) => (next) => (action) => {
  if (typeof action.then === 'function') {
    return action.then(next);
  }
  return next(action);
};
```
It is still a function that returns a function returning a function, but it's much easier to read, and the mental model you can use for this is that this is just a function with several arguments that are applied as they become available.

**configureStore.js**
```javascript
const logger (store) => (next) => {
  if (!console.group) {
    return next;
  }

  return (action) => { ... }
}
```
Finally, I'd like to fix the order in which `middlewares` are specified. They are currently specified in the order in which the dispatch function is overridden. However, it would be more natural to specify the order in which the action propagates through the `middlewares`.

This is why I'm changing my middleware declaration to specify them in the order in which the action travels through them. Instead, I will `wrapDispatchWithMiddlewares` from right to left, so I clone the past array and I reverse it.

**configureStore.js**
```javascript
const wrapDispatchWithMiddlewares = (store, middlewares) => {
  middlewares.slice().reverse().forEach(middleware => 
    store.dispatch = middleware(store)(store.dispatch)
  )
}
```
Let's recap what we learned about middleware so far. Middleware can serve very different purposes. It might **add some useful debugging information**, such as the case with `login` middleware, or it might **amend dispatch** to understand something other than plain actions, which is the case with the `promise` middleware.

**configureStore.js**
```javascript
const logger = (store) => (next) => {
  if (!console.group) {
    return next;
  }
}

const promise = (store) => (next) => (action) => {
  if (typeof action.then === 'function') {
    return action.then(next);
  }
  return next(action);
};
```
We created a new function that takes the `store` and an array of middlewares, and does the dirty work of reassigning the `store.dispatch` function. This lets me keep configure store a little bit more declarative, because I specified the middlewares in an array, and I call `wrapDispatchWithMiddlewares` to apply them.

**configureStore.js**
```javascript
const wrapDispatchWithMiddlewares = (store, middlewares) => {
  middlewares.slice().reverse().forEach(middleware => 
    store.dispatch = middleware(store)(store.dispatch)
  )
}

const configureStore = () => {
  const store = createStore(todoApp);
  const middlewares = [promise];

  if (process.env.NODE_ENV !== 'production') {
    middlewares.push(logger);
  }

  wrapDispatchWithMiddlewares(store, middlewares);

  return store;
};
```
Let's recap how `wrapDispatchWithMiddlewares` works internally. The `store` object that the application uses contains a dispatch function. `wrapDispatchWithMiddlewares` overrides it multiple times as it enumerates the middleware's array.

However, it reverses the array before enumerating it, so the final value of `store.dispatch` will correspond to the result of calling the first middleware in the array, which is the `promise` middleware.

**configureStore.js**
```javascript
const wrapDispatchWithMiddlewares = (store, middlewares) => {
  middlewares.slice().reverse().forEach(middleware => 
    store.dispatch = middleware(store)(store.dispatch)
  )
}
```
The `promise` middleware returns a dispatch function that understands promises, but to get it, we pass the `store` and the `next` dispatch function as curried arguments.

The `next` dispatch function is the one that we got from the middleware that appeared earlier in the reverse array, or later in the original array, and this is the logger middleware.

The logger middleware provided the dispatch function that logs the actions, but to get it, we passed the store and the next dispatch function to it, and it corresponded to the value `store.dispatch` that was there before the middleware was even applied, and this is the original dispatch implementation from create store.

The purpose of the `middlewares` is to **replace** the single dispatch function with a **chain of composable dispatch functions** which each can do something with an action.

The `promise` middleware appears before the `logger` middleware in the chain, and this is why the next inside it corresponds to the dispatch function returned from the `logger` middleware.

`Logger` middleware itself is the last middleware in chain, so its next function corresponds to the original `store.dispatch` function, which is defined inside create store.

Middleware is a powerful system that lets us put **custom behavior** before action reaches the reducers. People use middleware for different purposes, such as login, analytics, error-handling, asynchronous counterflow, and more.