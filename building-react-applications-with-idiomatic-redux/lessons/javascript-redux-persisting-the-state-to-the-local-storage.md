I want to persist the state of the application in the `localStorage` using **browser localStorage API**. I'm going to write a function called `loadState`  and a new model that I'm going to call `localStorage` as that is going to use the localStorage browser API.

The `loadState` function is going to look into `localStorage` by key, retrieve a string, and try to parse it as **JSON**. It's important that we wrap this code into `try`/`catch` because calls to your `localStorage.getItem` can fail if the user privacy mode does not allow the use of localStorage.

**localStorage.js**
```javascript
export const loadState = () => {
  try {
    const serializedState = localStorage.getItem('state');

  }
};
```

If the `serializedState` is **null** it means that the SKI does exist so I'll return `undefined` to let the reducers initialize the state instead. However if the `serializedState` string exists I'm going to use `JSON.parse` in order to turn it into the state object. Finally in case of any errors I'm going to play it safe and return `undefined` to let reducers initialize the application.

**localStorage.js**
```javascript
export const loadState = () => {
  try {
    const serializedState = localStorage.getItem('state');
    if (serializedState === null) {
      return undefined;
    }
    return JSON.parse(serializedState);
  } catch (err) {
    return undefined;
  }
};
```

Now that I have wrote the function to load the state, I might as well write a function that saves the state to localStorage. It's going to accept the `state` as an argument, and it's going to do the exact opposite thing.

**localStorage.js**
```javascript
export const saveState = (state) => {};
```

First it serializes it to string by using `JSON.stringify`. This will only work if the state is serializable, but this is the general recommendation in Redux. Your state should be serializable. Now that both `JSON.stringify` and `localStorage.setItem` can fail so it's important that we catch any errors to prevent the app from crashing.

**localStorage.js**
```javascript
export const saveState = (state) => {
  try {
    const serializedState = JSON.stringify(state);
    localStorage.setItem('state', serializedState);
  } catch (err) {
    // Ignore write errors.
  }
};
```


I'm just going to ignore them, but I might as well log them somewhere. Now I am also importing the `saveState` function I just wrote in the **index.js**. I need to save the state any time the store state changes, so I'm using the `store.subscribe` method to add a **listener** that will be invoked on any state change, and I'm passing the current state of the store to my `saveState` function.

**index.js**
```javascript
store.subscribe(() => {
  saveState(store.getState());
});
```

Let's see if it worked. I'm adding a few todos. I'm toggling one of the todos, and then I'm refreshing. The state of the app is preserved across reloads, and in fact the `visibilityFilter` is also preserved, which is probably not what we want, because usually we want to persist just the data and not the UI state.


![output](https://res.cloudinary.com/dg3gyk0gu/image/upload/v1553542110/transcript-images/javascript-redux-refactoring-the-entry-point-output.jpg)


To fix this rather than pass the whole state object I'll just pass an object with the `todos` field from the state object. This way if I start with a clean localStorage and I add a couple of todos, and then I toggle one of them and change the `visibilityFilter`, after refresh the todos are still there, but the `visibilityFilter` gets reset to all.


![output](https://res.cloudinary.com/dg3gyk0gu/image/upload/v1553542110/transcript-images/javascript-redux-refactoring-the-entry-point-output2.jpg)

**index.js**
```javascript
store.subscribe(() => {
  saveState({
    todos: store.getState().todos
  });
});
```

When the `store` is created the todos are preserved from the persisted state, but the `visibilityFilter` is initialized by the reducer. However, the current code has a bug. If I add a new todo to the existing todos it's not going to appear, and React is going to log a warning saying that I encountered two children with the same key, **zero**.


![error message](https://res.cloudinary.com/dg3gyk0gu/image/upload/v1553542110/transcript-images/javascript-redux-refactoring-the-entry-point-error.jpg)


What this means is that in the `TodoList` component when we render the todos, we use the `todo.id` as a key. The `todo.id` is assigned in the `addTodo` **action creator**, and it uses a local variable called `nextTodoId` as a counter.

**TodoList.js**
```javascript
const TodoList = ({ todos, onTodoClick }) => (
  <ul>
    {todos.map(todo =>
      <Todo
        key={todo.id}
        {...todo}
        onClick={() => onTodoClick(todo.id)}
      />
    )}
  </ul>
);
```

It's supposed to be unique. However if the application runs the second time the `nextTodoId` is going to be initialized to zero again so the new todo which is added also has an ID of zero just like the very first todo.

**actions/index.js**
```javascript 
export const addTodo = (text) => ({
  type: 'ADD_TODO', 
  id: (nextTodoID++).toString(),
  text,
});
```

To avoid problems like this I'm going to install an **npm** module called `node-uuid`. It is a very tiny module, and it exports a couple of functions. The function we're going to use is called `v4`, which is just a name of the standard.


![console output](https://res.cloudinary.com/dg3gyk0gu/image/upload/v1553542113/transcript-images/javascript-redux-refactoring-the-entry-point-console.jpg)


It generates a unique string ID every time, and we're going to use this ID instead of a counter. I'm replacing the counter declaration with an import of `v4` from the `node-uuid`, and I'm calling `v4` to generate a unique ID in my action creator.

**actions/index.js**
```javascript
import { v4 } from 'node-uuid';

export const addTodo = (text) => ({
  type: 'ADD_TODO',
  id: v4(),
  text,
});
```

Now let's run the app again with a clean localStorage. I'm adding a couple new todos, I'm toggling them, and these arrive refreshed in the app. I can change the `visibilityFilter`, but it doesn't get persisted, because we only want to persist the data.

Finally I'm adding a new todo, and it gets added successfully. It also gets persisted so I can refresh, do something with it, refresh, and it's there again in the correct state.


![todos](https://res.cloudinary.com/dg3gyk0gu/image/upload/v1553542110/transcript-images/javascript-redux-refactoring-the-entry-point-output3.jpg)


There is just one more thing left to do. We're currently call `saveState` inside the **subscribe listener** so it is called every time the storage state changes. However we would like to avoid calling it too often because it uses the expensive `stringify` operation.

**localStorage.js**
```javascript
export const saveState = (state) => {
  try {
    const serializedState = JSON.stringify(state);
    localStorage.setItem('state', serializedState);
  } catch (err) {
    // Ignore write errors.
  }
};

```

To solve this I'm going to use a library called **lodash** which includes a handy utility called `throttle`. Wrapping my call back in a `throttle` call insures that the inner function that I pass is not going to be called more often than the number of milliseconds I specify.

**index.js**
```javascript
store.subscribe(throttle(() => {
  saveState({
    todos: store.getState().todos
  });
}, 1000));

```

Now even if this store gets abated really fast we have a guarantee that we only write to the localStorage at most once a second. Now I'm adding the `import` for `throttle` from lodash and know that I imported directly from a file called **throttle** so that we don't end up with a whole lodash in our bundle just because of a single function.

**index.js**
```javascript
import throttle from 'lodash/throttle';
```

Let's recap how we added the localStorage persistence to the todos app. First we created the new module with `loadState` and `saveState` functions. `loadState` looks into the localStorage, and if there is a serialized string of our state it tries to parse it as **JSON**.

**localStorage.js**
```javascript
export const loadState = () => {
  try {
    const serializedState = localStorage.getItem('state');
    if (serializedState === null) {
      return undefined;
    }
    return JSON.parse(serializedState);
  } catch (err) {
    return undefined;
  }
};
```

If something goes wrong we return it `undefined` so that the app doesn't crash. We use the return value of `loadState` as the second argument to `createStore` so that it overrides the initial state specified by the reducers.

**index.js**
```javascript
const store = createStore(
  todoApp,
  persistedState
);
```

We want to be notified of the changes to the store state so we `subscribe` to it. We wrap our **subscriber** in `throttle` function from lodash in order to insure that it doesn't get called more often than once a second.

**index.js**
```javascript
store.subscribe(throttle(() => {
  saveState({
    todos: store.getState().todos
  });
}, 1000));
```

We only want to keep the `todos` and not the `visibilityFilter`, so we explicitly parse an object that contains just the `todos` from the current state. Inside `saveState` we're going to serialize the current state to string with `JSON.stringify` and try to `setItem`, but if something fails, we're just going to ignore this error so that the app doesn't crash.

**localStorage.js**
```javascript
export const saveState = (state) => {
  try {
    const serializedState = JSON.stringify(state);
    localStorage.setItem('state', serializedState);
  } catch (err) {
    // Ignore write errors.
  }
};
```

Finally we change the `ID` generation for `todos` from a simple in memory counter to a function from `node.uuid` that generates a unique ID string every time.

**actions/index.js**
```javascript
import { v4 } from 'node-uuid';

export const addTodo = (text) => ({
  type: 'ADD_TODO',
  id: v4(),
  text,
});
```
