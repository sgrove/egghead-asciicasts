Sometimes API requests fail, and I will simulate this by throwing inside the fake API client so that it returns a reject promise. If I run the app the loading indicator gets stuck because `isFetching` and flag get set to two, but there is no corresponding `RECEIVE_TODOS` action to set it to falls back again.

![console output](https://res.cloudinary.com/dg3gyk0gu/image/upload/v1553542111/transcript-images/javascript-redux-displaying-error-messages-output.jpg)

To fix this, I will start with the action creators file. First of all, I'll do a little cleanup. The `REQUEST_TODOS` action is never used outside of `fetchTodos` so I want to embed the object literal right inside fetch todos for clarity.

**actions/index.js**
```javascript
dispatch({
  type: `REQUEST_TODOS`,
  filter,
});
```

I will do the same for `RECEIVE_TODOS`. I will copy the ofbject literal for the `REVEIVE_TODOS` action and paste it where I dispatch it inside `fetchTodos` so it's easier to see the flow of `fetchTodos` action. I want to tweak the indentation a little bit to prepare for adding the second argument to promise.then method, the rejection handler.

**actions/index.js**
```javascript
return api.fetchTodos(filter).then(
  response => {
    dispatch({
      type: 'RECEIVE_TODOS',
      filter,
      response,
    });
  },
  error => {

  };
);
```

Now that the `fetchTodos` **action creator** dispatches quite a bit of actions, I want to rename them to a consistent naming scheme such as `FETCH_TODOS_REQUEST`, for requesting the todos, `FETCH_TODOS_SUCCESS`, for successful fetching the todos and `FETCH_TODO_FAILURE` for failing to fetch the todos.

I will pass two additional pieces of data, the `filter` and the error message, that I get the reading the error message field if specified or something went wrong as a fallback. The `fetchTodos` action creator now handles all the cases.

**actions/index.js**
```javascript
export const fetchTodos = (filter) => (dispatch, getState) => {
  if (getIsFetching(getState(), filter)) {...}

  dispatch({...})

  return api.fetchTodos(filter).then(
    response => {...},
    error => {
      dispatch({
        type: 'FETCH_TODOS_FAILURE',
        filter,
        message: error.message || 'Something went wrong.',
      });
    }
  );
};
```

I can also remove the old action creators that I have aligned into it. If I change the action types, I also need to change the corresponding reducers so I'm opening the `ids` reducer and rather than handle  it now needs to handle `FETCH_TODOS_SUCCESS`.

**createList.js**
```javascript
const createList = (filter) => {
  const ids = (state = [], action) => {
    if (filter !== action.filter) {
      return state;
    }
    switch (action.type) {
      case 'FETCH_TODOS_SUCCESS':
        return action.response.map(todo => todo.id);
      default:
        return state;
    }
  };
```

I also need to change the `isFetching` reducer. I'm renaming the `REQUEST_TODOS` the `FETCH_TODOS_REQUEST` and `RECEIVE_TODOS`, `FETCH_TODOS_SUCCESS` I also want to handle `FETCH_TODOS_FAILURE` and return `false` so that the loading indicator doesn't get stuck.

**createList.js**
```javascript
const isFetching = (state = false, action) => {
  if (filter !== action.filter) {
    return state;
  }
  switch (action.type) {
    case 'FETCH_TODOS_REQUEST':
      return true;
    case 'FETCH_TODOS_SUCCESS':
    case 'FETCH_TODOS_FAILURE':
      return false;
    default:
      return state;
  }
};
```

The last reducer I need to change is `byId`, and I just replace `RECEIVE_TODOS` with `FETCH_TODO_SUCCESS`.

**byId**
```javascript
const byId = (state = {}, action) => {
  switch (action.type) {
    case 'FETCH_TODOS_SUCCESS': // eslint-disable-line no-case-declarations
      const nextState = { ...state };
      action.response.forEach(todo => {
        nextState[todo.id] = todo;
      });
```

If I run the app now the loading indicator does not get stuck, because a corresponding failure action fires and so isFetching gets reset back to false again.

![console output two](https://res.cloudinary.com/dg3gyk0gu/image/upload/v1553542112/transcript-images/javascript-redux-displaying-error-messages-output2.jpg)

Let's display the error to the user. I have created a new file where I import react. I'm declaring a new functional stateless component called `FetchError` that takes the `message` as a prop, which is a string, and `onRetry` as a prop, which is a function.

**FetchError**
```javascript
import React from 'react';

const FetchError = ({ message, onRetry }) => (

);
```

The render deep will contain the children, an error saying that something bad happened including the `message` that is passed in the props, and a button that when clicked will invoke the `onRetry` callback prop so that the user can retry fetching the data.

**FetchError**
```javascript
import React from 'react';

const FetchError = ({ message, onRetry }) => (
  <div>
    <p>Could not fetch todos. {message}</p>
    <button onClick={onRetry}>Retry</button>
  </div>
);
```

I am exporting this component as a default export from `FetchError`, and I'm going to the `visibleTodoList` where I want to use it. I'm adding `FetchError` as an **import** to the `visibleTodoList`, and I'm scrolling down to the `render` method.

I will need the error message which I'm de-structuring from the props of `visibleTodoList` component, and inside the `render` method I will add another condition saying that if we have an error message in our props and we have no todos to display then I'm going to return the `FetchError` component.

**visibleTodoList.js**
```javascript
render() {
  const { isFetching, errorMessage, toggleTodo, todos } = this.props;
  if (isFetching && !todos.length) {
    return <p>Loading...</p>;
  }
  if (errorMessage && !todos.length) {
    return (
      <FetchError/>
    );
  }
```

The `FetchError` component itself wants a `message` prop which I can pass the `errorMessage` prop I just de-structured and an `onRetry` callback prop for which I will pass an **arrow function** that calls this `this.fetchData` to restart the data fetching process.

**visibleTodoList**
```html
<FetchError
  message={errorMessage}
  onRetry={() => this.fetchData()}
/>
```

In order to get the error message into the props I'll need to scroll down to my `mapStateToProps` implementation and put it there. I will use the same pattern as I currently do with `isFetching`, and I will get the error message prop by calling a selector called `getErrorMessage` passing the state of the app and the filter.

**VisibleTodoList**
```javascript
const mapStateToProps = (state, { params }) => {
  const filter = params.filter || 'all';
  return {
    isFetching: getIsFetching(state, filter),
    errorMessage: getErrorMessage(state, filter),
    todos: getVisibleTodos(state, filter),
    filter,
  };
};
```

The `getErrorMessage` selector does not exist yet so I need to scroll up to where I import the selectors from the root reducers file, and I will add `getErrorMessage` alongside `getIsFetching`. Inside the root reducers file I'm copy/paste in the `getIsFetching` selector, and I'm changing the name to get error message. It will also delegate to a selector with a same name defined in the createList.js.

**reducers/index.js**
```javascript
export const getIsFetching = (state, filter) =>
  fromList.getIsFetching(state.listByFilter[filter]);

export const getErrorMessage = (state, filter) =>
  fromList.getErrorMessage(state.listByFilter[filter]);
```

Inside createList.js, I'm adding a new exported selector called `getErrorMessage` that takes the state of the list and returns a property called `errorMessage`. Finally, I need to add a reducer managing the `errorMessage` field which I will add to the `combinedReducers` later.

**createList.js**
```javascript
export const getErrorMessage = (state) => state.errorMessage;
```

I'm declaring a new reducer called `errorMessage` with the initial state of `null`. **A reducer cannot have undefined initial state** so I have to make its absence explicit. Like in the other reducers in this file, I want to skip any actions with the filter that doesn't match the filter specified as an argument to `createList`.

**createList.js**
```javascript
const errorMessage = (state = null, action) => {
  if (filter !== action.filter) {
    return state;
  }
};
```

When the filter matches, I want to handle a few actions. I would like to display an error message whenever I get a failure so I handle a `FETCH_TODOS_FAILURE` by returning the message embedded the action.

**createList.js**
```javascript
const errorMessage = (state = null, action) => {
  if (filter !== action.filter) {...}
  switch (action.type) {
    case 'FETCH_TODOS_FAILURE':
      return action.message;
  }
};
```

If the user retries a request I want to clear the error message so I handle a fetch `FETCH_TODOS_REQUEST` and `FETCH_TODOS_SUCCESS` by returning `null`. Finally, for any other action, I'll just return the current state. The `errorMessage` reducer needs to be added to the `combineReducers` for the list so that its state becomes available on the `listState` object.

**createList.js**
```javascript
const errorMessage = (state = null, action) => {
  if (filter !== action.filter) {...}
  switch (action.type) {
    case 'FETCH_TODOS_FAILURE':
      return action.message;
    case 'FETCH_TODOS_REQUEST':
    case 'FETCH_TODOS_SUCCESS':
      return null;
    default:
      return state; 
  }
};

return combineReducers({
  ids,
  isFetching,
  errorMessage,
});
```



Now I will change my API layer so it doesn't throw every time. This way I'll get a chance to test how well the retry button works. If I run the app now, I can see the failure action being omitted with the corresponding message.

**api/index.js**
```javascript
export const fetchTodos = (filter) => 
  delay(500).then(() => {
    if (Math.random() > 0.5) {
      throw new Error('Boom!');
    }
```

![FETCH_TODOS_FAILURE](https://res.cloudinary.com/dg3gyk0gu/image/upload/v1553542111/transcript-images/javascript-redux-displaying-error-messages-output3.jpg)

Thanks to the new reducer, this message will now make its way into the next state inside `listByFilter`, `all`, `errorMessage` and so the component can display it. If I press the retry button, the request action will clear the error message and then the success action will populate the list of todos.

![FETCH_TODOS_SUCCESS](https://res.cloudinary.com/dg3gyk0gu/image/upload/v1553542112/transcript-images/javascript-redux-displaying-error-messages-output4.jpg)

Let's recap how we handle errors for every list independently. Inside the fake API client, I randomly throw an error so `fetchTodos` returns a rejected promise once in a while. Having separate action creators for request success and failure cases seemed like overkill. I in-lined the corresponding action objects right inside the async action creator.

**actions/index.js**
```javascript
dispatch({
  type: 'FETCH_TODOS_REQUEST',
  filter,
});

return api.fetchTodos(filter).then(
  response => {
    dispatch({
      type: 'FETCH_TODOS_SUCCESS',
      filter,
      response,
    });
  },
```

I also renamed them to `FETCH_TODOS_REQUEST`, `FETCH_TODOS_SUCCESS`, and I added a new `FETCH_TODOS_FAILURE` action so that we can reset the loading indicator and display the error message.

**actions/index.js**
```javascript
error => {
  dispatch({
    type: 'FETCH_TODOS_FAILURE',
    filter,
    message: error.message || 'Something went wrong.',
  });
}
```

I am passing the rejection handler as a second argument to the `promise.then` method. If the promise returned by the API gets rejected, this function will be called with the error as the argument.

You might have seen a different way of handling promise errors in some examples where only one argument is passed and then you call catch on the resulting promise. The downside of this approach is if one of your reducers or subscribe components throws while handling this action you'll get into the catch block and so you'll display an internal error message to the user.


**actions/index.js**
```javascript
return api.fetchTodos(filter).then(
  response => {
    dispatch({
      type: 'FETCH_TODOS_SUCCESS',
      filter,
      response,
    });
  },
).catch(
  error => {
    dispatch({
      type: 'FETCH_TODOS_FAILURE',
      filter,
      message: error.message || 'Something went wrong.',
    });
  }
)
```

To avoid this error, I recommend that you don't use `catch` in this scenario and just pass the second argument so it catches only the errors from the underlying API promise. The `error` object usually comes with a message that we can wrap or display directly to the user with a fallback.

**actions/index.js**
```javascript
error => {
  dispatch({
    type: 'FETCH_TODOS_FAILURE',
    filter,
    message: error.message || 'Something went wrong.',
  });
}
```

We make it available to the reducers as the `message` field on the action object. I renamed the action types so I also needed to update the reducer inside byId.js and the reducers inside createList.js to use the new names.

**byId.js**
```javascript
const byId = (state = {}, action) => {
  switch (action.type) {
    case 'FETCH_TODOS_SUCCESS': // eslint-disable-line no-case-declarations
```

**createList.js**
```javascript
const createList = (filter) => {
  const ids = (state = [], action) => {
    if (filter !== action.filter) {
      return state;
    }
    switch (action.type) {
      case 'FETCH_TODOS_SUCCESS':
```

We only want to update the IDs on a successful fetch. However, we want to update the indicator in all three cases. In particular, I reset the loading indicator both on success and on failure. I also added a new reducer that keeps track of the error message for the given tab.

**createList.js**
```javascript
const isFetching = (state = false, action) => {
  if (filter !== action.filter) {
    return state;
  }
  switch (action.type) {
    case 'FETCH_TODOS_REQUEST':
      return true;
    case 'FETCH_TODOS_SUCCESS':
    case 'FETCH_TODOS_FAILURE':
      return false;
    default:
      return state;
  }
};


const errorMessage = (state = null, action) => {...}
```

When the request fails, its state becomes the message from the action. But if it succeeds or if a new request starts, it is reset to `null`. Like in other reducers in this file, we are only handling actions that have the same filter as the filter this list was created with.

**createList**
```javascript
const errorMessage = (state = null, action) => {
  if (filter !== action.filter) {
    return state;
  }
  switch (action.type) {
    case 'FETCH_TODOS_FAILURE':
      return action.message;
    case 'FETCH_TODOS_REQUEST':
    case 'FETCH_TODOS_SUCCESS':
      return null;
    default:
      return state;
  }
};
```

The `return combineReducers` now manages both `isFetching` and the currently displayed error message. This lets me add a new selected called `getErrorMessage` that just reads the error message field from the combined state object.

**createList.js**
```javascript
export const getErrorMessage = (state) => state.errorMessage;
```

I'm calling this selector from the top level reducer file, and I'm passing the slice of the state corresponding to the list for the specified filter. The top level `getErrorMessage` selector will be called inside `mapStateToProps` of the `visibleTodoList` component.

**reducers/index.js**
```javascript
export const getErrorMessage = (state, filter) =>
  fromList.getErrorMessage(state.listByFilter[filter]);
```

It receives the global state and the current filter as arguments, and the result will be available as `errorMessage` prop inside the `visibleTodoList` components render function.

**VisibleTodoList.js**
```javascript
cosnt mapStateToProps = (state, { params }) => {
  const filter = params.filter || 'all';
  return {
    isFetching: getIsFetching(state, filter),
    errorMessage: getErrorMessage(state, filter),
    todos: getVisibleTodos(state, filter),
    filter,
  };
};
```

I'm de-structuring the `errorMessage` from the props, and if I have an `errorMessage` and no todo list to show I will render the `FetchError` component passing the `errorMessage` and a function that refetches the data as the props.

**VisibleTodoList.js**
```javascript
render() {
  const { isFetching, errorMessage, toggleTodo, todos } = this.props;
  if (isFetching && !todos.length) {
    return <p>Loading...</p>;
  }
  if (errorMessage && !todos.length) {
    return (
      <FetchError
        message={errorMessage}
        onRetry={() => this.fetchData()}
      />
    );
  }
```

Finally, I created a new component called `FetchError` that displays the message from the props and renders a button that lets the user retry the fetch action.

**FetchError.js**
```javascript
const FetchError = ({ message, onRetry }) => (
  <div>
    <p>Could not fetch todos. {message}</p>
    <button onClick={onRetry}>Retry</button>
  </div>
);
```
