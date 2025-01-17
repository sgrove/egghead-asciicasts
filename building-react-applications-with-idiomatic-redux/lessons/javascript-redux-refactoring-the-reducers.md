Earlier, we removed the `visibilityFilter` reducer, and so the root reducer in the app now combines only a single `todos` reducer. Since `index.js` acts effectively as a proxy to the **todos reducer**, I will just remove `index.js` completely, and I will rename `todos.js` to `index.js`, thereby making the todos my new root reducer.

The root reducer file now contains `byId`, `allIds`, `activeIds`, and `completedIds`, and I'm going to extract some of them into separate files. I'm starting with `byId` reducer, and I'm creating the file called `byId.js`, where I paste this reducer and export it as a [default export](https://egghead.io/lessons/ecmascript-6-es6-modules-es2015-import-and-export?course=learn-es6-ecmascript-2015#/tab-transcript).

**byId.js**
```javascript
const byId = (state = {}, action) => {
  switch (action.type) {
    case 'RECEIVE_TODOS':
    const nextState = { ...state };
    action.response.forEach(todo => {
      nextState[todo.id] = todo;
    });
    return nextState;
  default:
    return state;
  }
}

export default byId;
```
I'm also adding a named export for a selector called `getTodo`, that takes the `state` and `id`, where the `state` corresponds to the state of `byId` reducer. 

**byId.js**
```javascript
export const getTodo = (state, id) => state[id];
```
Now I'm going back to my `index.js` file, and I can import the reducer as a default import, and I can import any associated selectors in a single object with a namespace import.

**index.js**
```javascript
import byId, * as fromById from './byId';
```
If we take a look at the reducers managing the IDs, we will notice that their code is almost exactly the same except for the `filter` value which they compare action filter to. I will create a new function called `createList` that takes filter as an argument.

It returns another function, a reducer that handles the IDs for the specified filter, so its state shape is an array, and I can copy-paste the implementation from `allIds`. I just need to change the `all` literal here to the `filter` argument to the `createList` function, so that we can create it for any filter.

**index.js**
```javascript
const createList = (filter) => {
  return (state = [], action) => {
    if (action.filter !== filter) {
      return state;
    }
    switch (action.type) {
      case 'RECEIVE_TODOS':
        return action.response.map(todo => todo.id);
      default:
        return state;
    }
  };
};
```
Now I can remove the `allIds`, `activeIds`, and `completedIds` reducer code completely. Instead, I will generate the reducers using the new `createList` function I wrote, and passing the `filter` as an argument to it.

**index.js**
```javascript
const idsByFilter = combineReducers({
  all: createList('all'),
  active: createList('active'),
  completed: createList('completed'),
})
```
Next, I will extract the `createList` function into a separate file, just like I extracted the `byId` reducer. I added a new file called `createList.js`. I pasted my `createList` function there, and I will export it as a default export. Now that it's in a separate file, I'm adding a public API for accessing the `state` in form of a selector. That is called `getIds`, and for now, it just returns the state of the list, but this may change in the future.

**createList.js**
```javascript
const createList = (filter) => {
  return (state = [], action) => {
    if (action.filter !== filter) {
      return state;
    }
    switch (action.type) {
      case 'RECEIVE_TODOS':
        return action.response.map(todo => todo.id);
      default:
        return state;
    }
  };
};

export default createList;

export const getIds = (state) => state;
```
Now I will go back to my `index.js` file, and I will import `createList` just like I usually import reducers, and I will import any named selectors from this file, as well.

**index.js**
```javascript
import creatList, * as fromList from './createList';
```
I'm renaming the `Ids` by filter reducer to `listByFilter`, because now that the list implementation is in a separate file, I'll consider its `state` structure to be opaque, and to get the list of IDs, I will use the get IDs selector that it exports.

**index.js**
```javascript
const idsByFilter = combineReducers({
  all: createList('all'),
  active: createList('active'),
  completed: createList('completed'),
})

const todos = combineReducers({
  byID,
  listByFilter,
});

export const getVisibleTodos = (state, filter) => {
  const ids = fromList.getIds(state.listByFilter[filter]);
  return ids.map(id => fromById.getTodo(state.byId[id]);
};
```
Since I also moved the `byId` reducer into a separate file, I also don't want to make an assumption that it's just a lookup table, and I will use `fromById.getTodo` selector that it exports and pass its `state` and the corresponding `id`.

This lets me change the state shape of any reducer in the future without rippling changes across the code base. Let's recap how we refactored the reducers.

First of all, the `todos` reducer is now the root reducer of the application, and its file has been renamed to `index.js`. 

**index.js**
```javascript
const todos = combineReducers({
  byID,
  listByFilter,
});
```
We extracted the `byId` reducer into a separate file. The `byId` reducer is now declared here, and it is exported from this module as a default export. 

**byid.js**
```javascript
const byId = (state = {}, action) => {
  switch (action.type) {
    case 'RECEIVE_TODOS':
    const nextState = { ...state };
    action.response.forEach(todo => {
      nextState[todo.id] = todo;
    });
    return nextState;
  default:
    return state;
  }
}

export default byId;
```
To encapsulate the knowledge about the state shape in this file, we export a new selector that just gets the `todo` by its ID from the lookup table.

**byid.js**
```javascript
export const getTodo = (state, id) => state[id];
```
We also created a new function called `createList` that we use to generate the reducers, managing the lists of fetched `todos` for any given `filter`.

**byid.js**
```javascript
const idsByFilter = combineReducers({
  all: createList('all'),
  active: createList('active'),
  completed: createList('completed'),
})
```
Inside `createlist.js`, we have a `createList` function that takes filter as an argument, and it returns a reducer that manages the fetched IDs for this filter.

The generated reducers will handle the `RECEIVE_TODOS` action, but they will keep any action that has the `filter` different from the one they were created with.

**createList.js**
```javascript
const createList = (filter) => {
  return (state = [], action) => {
    if (action.filter !== filter) {
      return state;
    }
    switch (action.type) {
      case 'RECEIVE_TODOS':
        return action.response.map(todo => todo.id);
      default:
        return state;
    }
  };
};
```
`CreateList` is the default export from this file, but we also export a selector that gets the ids from the current state. Right now, the ids are the current state, but we are free to change this in the future.

In `index.js`, we use the namespace import syntax to grab all selectors from the corresponding file into an object. 

**index.js**
```javascript
import { combineReducers } from 'redux';
import byId, * as fromById from './byId';
import creatList, * as fromList from './createList';
```
The `listByFilter` reducer combines the reducers generated by `createList`, and it uses the filters as the keys.

**index.js**
```javascript
const listByFilter = combineReducers({
  all: createList('all'),
  active: createList('active'),
  completed: createList('completed'),
})
```
Because `listByFilter` is defined in this file, the `getVisibleTodo` selector can make assumptions about its `state` shape and access it directly. However, the implementation of `createList` is in a separate file, so this is why it uses the `fromList.getIds` selector to get the IDs from it.

**index.js**
```javascript
export const getVisibleTodos = (state, filter) => {
  const ids = fromList.getIds(state.listByFilter[filter]);
  return ids.map(id => fromById.getTodo(state.byId[id]);
};
```
Now that we have the IDs, we want to map them to the todos inside the `state.byId`. But now that it's in a separate file, rather than reach into it directly, we use the selector that it exports to access the individual todo.

Redux does not enforce that you encapsulate the knowledge about the state shape in particular reducer files. However, it's a nice pattern, because it lets you change the state that is stored by reducers without having to change your components or your tests if you use selectors together with reducers in your tests.