# SpaceAce

A fancy immutable storage library for JavaScript

[![Build Status](https://travis-ci.org/JonAbrams/SpaceAce.svg?branch=master)](https://travis-ci.org/JonAbrams/SpaceAce)

## Intro

SpaceAce is used to store the _state_ of your front-end application. Think of it as a replacement for `this.setState(…)` in your React apps.

The goal is a library that is as powerful and useful as Redux, but more modular with an easier to use API.

You may have heard of Flux and/or Redux, SpaceAce is neither, it's a new idea (AFAIK).

Any and all [feedback is welcome](https://twitter.com/JonathanAbrams)!

## Background

It's become standard practice in front-end JavaScript to have views that receive data via a one-way data flow. This was popularized by React and Flux/React. The idea is that the view components contain no state. Instead, they receive their data via properties given by parent components, which ultimately is given data from a single data store. This store's _state_ object is immutable. Instead of being changed, it's replaced with a slightly altered copy, and the new state is passed down through the components. This state is generated by a function known as a _reducer_. The reducer receives an action, and based on the action it takes the existing state and generates a new one. This new state is then passed to the view components, triggering a re-render of them.

This concept, referred to as _Flux_ has many advantages:

- It's much easier to reason through the changes your application experiences.
- You can easily log changes.
- You can replay/undo state changes.
- Has great libraries such as _Redux_ that implement it.

That all sounds nice but it has its problems:

- It's not clear which parts of the store are relevant to each view component.
- It's hard to architect the application code around this concept. People often put all their actions in a single `actions.js`, and end up with a giant reducer in `reducer.js`. This goes against the goal of keeping all the code relevant to a component in a single file.
- This pattern results in a lot of boilerplate code. Actions are defined in one file, referenced multiple times in the same file, and then referenced in the app reducer. It'd be nice to define an action once, in the component that uses it, and then just call it. It'd be even nicer if you didn't have to explicitly name the action.

If you never used Redux and didn't understand that, don't worry, you don't need to with SpaceAce.

SpaceAce is designed to provide the above benefits without the above downsides.

## Example Usage

SpaceAce can be used with any front-end view library (such as Vue and Angular), but the examples below are done with React.

**index.js**
```jsx
import react from 'react';
import ReactDOM from 'react-dom';
import Space from 'spaceace';
import Container from './Container';

// Create the root "space" along with its initial state
const rootSpace = new Space({ name: 'Jon', todos: [] });
rootSpace.subscribe(causedBy => {
  // Example `causedBy`s:
  // 'todoList#addTodo', 'todos[akd4a1plj]#toggleDone'
  console.log(`Re-render of <Container /> caused by ${causedBy}`);
  ReactDOM.render(
    <Container space={rootSpace} />,
    document.getElementById('react-container')
  );
});
```

**Container.js**
```jsx
import react from 'react';
import TodoList from './TodoList';

export default function Container({ space }) {
  const state = space.state;

  return (
    <div>
      <h1>Welcome {state.name}</h1>
      <TodoList
        // subSpace takes over `state.todos`, turning it into a child space
        space={space.subSpace('todos')}
        name={state.name}
      />
    </div>
   );
```

**TodoList.js**
```jsx
import react from 'react';
import uuid from 'uuid/v4';
import Todo from 'Todo';

export default function TodoList({ space, name }) {
  const todos = space.state;

  return(
    <h2>{name}'s Todos:</h2>
    <button onClick={space.setState(addTodo)}>Add Todo</button>
    <ul className='todos'>
      {todos.map(todo =>
        <Todo space={space.subSpace(todo.id)} />
      )}
    </ul>
  );
};

// setState callbacks are given the space first, then the event.
// If the space is an array, the result overwrites the existing state
function addTodo(space, e) {
  const todos = space.state;

  e.preventDefault();

  return [
      // All items that exist in a list, like this one, need a unique 'id'
      // for when they are later accessed as a subSpace
      // uuid() is a handy module for generating a unique 'id'
      { id: uuid(), msg: 'A new TODO', done: false }
     ].concat(todos)
   };
 }
}
```

**Todo.js**
```jsx
import react from 'react';

export default function Todo({ space }) {
  const todo = space.state; // The entire state from this space is the todo
  const { setState } = space;
  const doneClassName = todo.done ? 'done' : '';

  return(
    <li className='todo'>
      <input type='checkbox' checked={done} onChange={setState(toggleDone)} />
      <span className={doneClassName}>{todo.msg}</span>
      <button onClick={setState(removeTodo)}>Remove Todo</button>
    </li>
  );
};

// The returned value from an action is merged onto the existing state
// In this case, only the `done` attribute is changed on the todo
function toggleDone({ state: todo }, e) {
  return { done: !todo.done };
}

// Returning null causes the space to be removed from its parent
function removeTodo(todoSpace, e) {
  e.preventDefault();

  return null;
}
```

## Documentation

### What is a Space?

`Space` is the default class provided by the `spaceace` npm package.

Every `space` consists of:
- `state`: An immutable state, which can only be overwritten using an action.
- `setState`: A method for changing a space's state. Accepts an object or function.
- `subSpace`: A method for spawning child spaces.
- `subscribe`: A method for subscribing to updates.
- `parentSpace`: A method for getting a parent space with a specified name.

You create a new space by calling `new Space(…)` e.g.
```javascript
const rootSpace = new Space({ initialState: true, todoList: { todos: [] } });
```

### state

`state` is a getter method on every space. It generates a frozen/immutable object
that can be used to render a view. It includes the state of any child spaces as well. Do not try to change it directly.


### setState

Parameters:
  - Object (for merging onto state) or a function (as a callback)
  - (optional) String – Used as a name for logging

Given an object, `setState` merges it into the space's state. Pass in a second parameter
to give the update a name, for logging purposes.

Given a function, `setState` wraps it and _returns a function_. This new function can then be used as an event handler.

The provided callback function is called with the space as the first parameter. This saves you from needing to bind `this` to your event handlers.

If `null` is returned from a space that is a list item, then the space is removed
from the list it is in.

The event is passed in as the second parameter to the callback, useful for calling `event.preventDefault()`, or for reading `event.target.value`.

e.g. Given the following React component:
```jsx
class SignupModal extends React.Component {
  closeModal(space, event) {
    event.preventDefault();
    return { isOpen: false };
  }

  toggleMinify({ state }, event) {
    event.preventDefault();
    return { isMinified: !state.isMinified };
  }

  render() {
    const { state, setState, subSpace } = this.props;
    return (
      <Modal isOpen={state.isOpen}>
        <SignupForm {...subSpace('signupForm')} />
        <a href='#' onClick={setState(this.closeModal)}>&times;</a>
        <a href='#' onClick={setState(this.toggleMinify)}>
          {state.isMinified ? 'Expand' : 'Shrink'}
        </a>
      </Modal>
    )
  }
}
```

### subSpace

One of the main feature of SpaceAce is the ability to break up a store into individual sub-stores, or spaces. Any space, whether it's a root or a child, can have child spaces.

When a child space's state is updated, it notifies its parent space, which causes it to update its state (which includes the child's state) and notifies its subscribers, and then notifies its parent space, and so on.

### subscribe

Registers a callback function to be called whenever the space's state is updated.
This includes if a child space's state is updated.

It calls the subscriber with a single parameter: `causedBy`, which specifies why
the subscriber was invoked. It's useful for debugging purposes. The format of
`causedBy` is `spaceName#functionName`.

Note: For convenience, this subscriber is called immediately when it's declared, with _causedBy_ set to `'initialized'`.

e.g.
```jsx
const userSpace = new Space({ name: 'Jon' });

userSpace.subscribe(causedBy => {
  console.log('Re-rendered by: ', causedBy);
  ReactDOM.render(
    <Component {...userSpace} />,
    document.getElementById('react-container')
  );
});
```

### parentSpace

Example: `space.parentSpace('root')` or `space.parentSpace('todos')`

Ideally, you shouldn't need to access a parent space, but if you do, this will return the space with the specified name. It searches up the state tree until it finds the space with the specified name. The root space is given the default name `'root'`. If no matching space is found, `null` is returned.

#### Spawning Children

Calling `subSpace` with a string will turn that attribute of a space into a child space.

e.g. Given a space called `userSpace` with this state:
```javascript
{
  name: 'Jon',
  settings: {
    nightMode: true,
    fontSize: 12
  }
}
```

You can convert the `settings` into a child space with `userSpace.subSpace('settings')`.

Note that even though `settings` is now a space, the state of `userSpace` hasn't changed.
At least not until the `settings` space is updated with a change.

## FAQ

**What's the difference between a state and a space?**

Think of state as an object with a bunch of values. A space contains that state, but
provides a few handy methods meant to interact with it. So if you're in a component
that doesn't need to change the state's contents, nor does it need to spawn subspaces,
then you don't need to give it a space, you can just give it the state.

**How do I add middleware like in Redux?**

Hopefully that feature will come in v2!

**Are spaces immutable?**

Sort of. The state you get from a space is an immutable object. You cannot change it
directly, if you do so you may get an error (if 'use strict' is enabled). But… you can
change it using the `setState` function provided by the state's space.

**Why do list items need an `id` key?**

Due to the fact that the state is immutable, if a sub-space for an item wants to update
its state, SpaceAce needs to find it in the parent space's state. The way we've
solved this is to use a unique id field.

**Why do I need to specify a name to get a parent space?**

One of the goals of SpaceAce is to keep components as modular/reusable as possible.
The use of `parentSpace` risks that by requiring a component to have a known parent
component. By requiring a name to be specified, it encourages you to consider the
relationship and interaction contract between the components. But perhaps more importantly,
it also allows you to move the component around your app's structure and not have
it break in an odd way.

## License

MIT

## Author

Created with the best of intentions by [Jon Abrams](https://twitter.com/JonathanAbrams)
