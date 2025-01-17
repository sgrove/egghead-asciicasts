In the next lessons, we won't be dealing with persistence anymore. Instead, we're going to add some **asynchronous** fetching to the app. This is why I'm removing all the code related to the `localStorage` persistence, and I'm going to delete the localStorage.js, as well.

I also added a new module that implements a fake remote API. It's not really remote. I keep all the todos in memory, but I also added an artificial `delay`. I have methods that return promises just like a real API implementation. But instead, it just pretends that it calls the server and returns the todos from the in memory database.

**src/api/index.js**
```javascript
const delay = (ms) =>
  new Promise(resolve => setTimeout(resolve, ms));

export const fetchTodos = (filter) =>
  delay(500).then(() => {
    switch (filter) {
      case 'all':
        return fakeDatabase.todos;
      case 'active':
        return fakeDatabase.todos.filter(t => !t.completed);
      case 'completed':
        return fakeDatabase.todos.filter(t => t.completed);
      default:
        throw new Error(`Unknown filter: ${filter}`);
    }
  });
```

This approach lets us explore how Redux works with asynchronous data fetching without writing a real backend for the app. Now I can open any other module of my app and I can import fetchTodos from the API module.

**src/index.js**
```javascript
import { fetchTodos } from './api';
```

We will learn how to put these todos into the Redux store later. But for now, let's just make sure that calling `fetchTodos()` with a `filter` argument returns a promise that results through an array of todos just like a REST backend would return an array.

The fake API waits for half a second to simulate the network connection, and then resolves the promise to an array of todos that we will treat as if they were retrieved from a remote server.

![console output](https://res.cloudinary.com/dg3gyk0gu/image/upload/v1553542111/transcript-images/javascript-redux-adding-a-fake-backend-to-the-project-output.jpg)