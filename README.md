# ToDo

This is a simple to do list application built using React.js and Redux.js for learning purposes. Feel free to follow through with this ReadME file to understand the different aspects of the Redux.js library.

# Actions

Actions are payloads of information that send data from your application to your store. They are the only source of information for the store. You send them to the store using *store.dispatch()*.

Actions are plain JS objects. Actions must have a type property that indicates the type of action being performed. Types should typically be defined as string constants. Once your app is large enough, you may want to move them into a separate module.

# Action Creators

Action creators are exactly that - functions that create actions. Its easy to conflate the terms "action" and "action creator", so do your best to use the proper term.

In Redux, action creators simply return an action, this makes them portable and easy to test.
In traditional Flux, action creators often trigger a dispatch when invoked. In Redux this is not the case. Instead, to actually initiate a dispatch, pass the result to the dispatch() function. Alternatively you can create a bound action creator that automatically dispatches those actions.

The dispatch() function can be accessed directly from the store as store.dispatch(), but more likely you'll access it using a helper like react-redux's connect(). You can use bindActionCreators() to automatically bind many action creators to a dispatch() function.

# Reducers

Reducers specify how the application's state changes in response to actions sent to the store. Remember that actions only describe what happened, but don't describe how the application's state changes.

# Designing the State Shape

In Redux, all the application state is stored as a single object. Its a good idea to think of its shape before writing any code. What's the minimal representation of your app's state as an object?

In Redux, all the application state is stored as a single object. It's a good idea to think of its shape before writing any code. What's the minimal representation of your app's state as an object?

For our todo app, we want to store two different things:

- The currently selected visibility filter.
- The actual list of todos.

You'll often find that you need to store some data, as well as some UI state, in the state tree. This is fine, but try to keep the data separate from the UI state.

# Note on Relationships

In a more complex app, you're going to want different entities to reference each other. We suggest that you keep your state as normalized as possible, without any nesting. Keep every entity in an object stored with an ID as a key, and use IDs to reference it from other entities, or lists. Think of the app's state as a database. This approach is described in normalizr's documentation in detail. For example, keeping todosById: { id -> todo } and todos: array<id> inside the state would be a better idea in a real app, but we're keeping the example simple.

# Handling Actions

Now that we've decided what our state object looks like, we're ready to write a reducer for it. The reducer is a pure function that takes the previous state and an action, and returns the next state.

  *(previousState, action) => newState*

It's called a reducer because it's the type of function you would pass to Array.prototype.reduce(reducer, ?initialValue). It's very important that the reducer stays pure. Things you should never do inside a reducer:

- Mutate its arguments;
- Perform side effects like API calls and routing transitions;
- Call non-pure functions, e.g. Date.now() or Math.random().

Just remember that the reducer must be pure. Given the same arguments, it should calculate the next state and return it. No surprises. No side effects. No API calls. No mutations. Just a calculation.

Now that we've written a boilerplate intial reducer, lets handle SET_VISIBILITY_FILTER. All it needs to do is to change visibilityFilter on the state. Easy

Note that:

We don't mutate the state. We create a copy with Object.assign(). Object.assign(state, { visibilityFilter: action.filter }) is also wrong: it will mutate the first argument. You must supply an empty object as the first parameter. You can also enable the object spread operator proposal to write { ...state, ...newState } instead.

We return the previous state in the default case. It's important to return the previous state for any unknown action

# Handling More Actions

We have two more actions to handle! Just like we did with SET_VISIBILITY_FILTER, we'll import the ADD_TODO and TOGGLE_TODO actions and then extend our reducer to handle ADD_TODO.

Just like before, we never write directly to state or its fields, and instead we return new objects. The new todos is equal to the old todos concatenated with a single new item at the end. The fresh todo was constructed using the data from the action.

Because we want to update a specific item in the array without resorting to mutations, we have to create a new array with the same items except the item at the index. If you find yourself often writing such operations, it's a good idea to use a helper like immutability-helper, updeep, or even a library like Immutable that has native support for deep updates. Just remember to never assign to anything inside the state unless you clone it first.

# Splitting Reducers

Is there a way to make it easier to comprehend? It seems like todos and visibilityFilter are updated completely independently. Sometimes state fields depend on one another and more consideration is required, but in our case we can easily split updating todos into a separate function, and we've done exactly that.

Note that todos also accepts state—but it's an array! Now todoApp just gives it the slice of the state to manage, and todos knows how to update just that slice. This is called reducer composition, and it's the fundamental pattern of building Redux apps.

Let's explore reducer composition more. Can we also extract a reducer managing just visibilityFilter? We can.

Below our imports, let's use ES6 Object Destructuring to declare SHOW_ALL:

  *const { SHOW_ALL } = VisibilityFilters*

Then:

   *function visibilityFilter(state = SHOW_ALL, action) {
      switch (action.type) {
        case SET_VISIBILITY_FILTER:
          return action.filter
        default:
          return state
      }
    }*

Now we can rewrite the main reducer as a function that calls the reducers managing parts of the state, and combines them into a single object. It also doesn't need to know the complete initial state anymore. It's enough that the child reducers return their initial state when given undefined at first.

   *function todos(state = [], action) {
      switch (action.type) {
        case ADD_TODO:
          return [
            ...state,
            {
              text: action.text,
              completed: false
            }
          ]
        case TOGGLE_TODO:
          return state.map((todo, index) => {
            if (index === action.index) {
              return Object.assign({}, todo, {
                completed: !todo.completed
              })
            }
            return todo
          })
        default:
          return state
      }
    }
    ​
    function visibilityFilter(state = SHOW_ALL, action) {
      switch (action.type) {
        case SET_VISIBILITY_FILTER:
          return action.filter
        default:
          return state
      }
    }
    ​
    function todoApp(state = {}, action) {
      return {
        visibilityFilter: visibilityFilter(state.visibilityFilter, action),
        todos: todos(state.todos, action)
      }
    }*

Note that each of these reducers is managing its own part of the global state. The state parameter is different for every reducer, and corresponds to the part of the state it manages.

This is already looking good! When the app is larger, we can split the reducers into separate files and keep them completely independent and managing different data domains.

Finally, Redux provides a utility called combineReducers() that does the same boilerplate logic that the todoApp above currently does. With its help, we can rewrite todoApp like this:

   *import { combineReducers } from 'redux'
    ​
    const todoApp = combineReducers({
      visibilityFilter,
      todos
    })
    ​
    export default todoApp*

All combineReducers() does is generate a function that calls your reducers with the slices of state selected according to their keys, and combining their results into a single object again. It's not magic. And like other reducers, combineReducers() does not create a new object if all of the reducers provided to it do not change state.

- Note for ES6 Savvy Users:

Because combineReducers expects an object, we can put all top-level reducers into a separate file, export each reducer function, and use import * as reducers to get them as an object with their names as the keys:

   *import { combineReducers } from 'redux'
    import * as reducers from './reducers'
    ​
    const todoApp = combineReducers(reducers)*

# Structuring Reducers

At its core, Redux is really a fairly simple design pattern: all your "write" logic goes into a single function, and the only way to run that logic is to give Redux a plain object that describes something that has happened.  The Redux store calls that write logic function and passes in the current state tree and the descriptive object, the write logic function returns some new state tree, and the Redux store notifies any subscribers that the state tree has changed.

Redux puts some basic constraints on how that write logic function should work.  As described in Reducers, it has to have a signature of (previousState, action) => newState, is known as a reducer function, and must be pure and predictable.

Beyond that, Redux does not really care how you actually structure your logic inside that reducer function, as long as it obeys those basic rules.  This is both a source of freedom and a source of confusion.  However, there are a number of common patterns that are widely used when writing reducers, as well as a number of related topics and concepts to be aware of.  As an application grows, these patterns play a crucial role in managing reducer code complexity, handling real-world data, and optimizing UI performance.

# Basic Reducer Structure

First and foremost, it's important to understand that your entire application really only has one single reducer function: the function that you've passed into createStore as the first argument.  That one single reducer function ultimately needs to do several things:

The first time the reducer is called, the state value will be undefined.  The reducer needs to handle this case by supplying a default state value before handling the incoming action.

It needs to look at the previous state and the dispatched action, and determine what kind of work needs to be done

Assuming actual changes need to occur, it needs to create new objects and arrays with the updated data and return those

If no changes are needed, it should return the existing state as-is.

The simplest possible approach to writing reducer logic is to put everything into a single function declaration, like this:

   *function counter(state, action) {
      if (typeof state === 'undefined') {
        state = 0; // If state is undefined, initialize it with a default value
      }
    ​
      if (action.type === 'INCREMENT') {
        return state + 1;
      }
      else if (action.type === 'DECREMENT') {
        return state - 1;
      }
      else {
        return state; // In case an action is passed in we don't understand
      }
    }*

Notice that this simple function fulfills all the basic requirements.  It returns a default value if none exists, initializing the store; it determines what sort of update needs to be done based on the type of the action, and returns new values; and it returns the previous state if no work needs to be done.

There are some simple tweaks that can be made to this reducer.  First, repeated if/else statements quickly grow tiresome, so it's very common to use switch statements instead.  Second, we can use ES6's default parameter values to handle the initial "no existing data" case.  With those changes, the reducer would look like:

   *function counter(state = 0, action) {
      switch (action.type) {
        case 'INCREMENT':
          return state + 1;
        case 'DECREMENT':
          return state - 1;
        default:
          return state;
      }
    }*

This is the basic structure that a typical Redux reducer function uses.

# Basic State Shape

Redux encourages you to think about your application in terms of the data you need to manage.  The data at any given point in time is the "state" of your application, and the structure and organization of that state is typically referred to as its "shape".  The shape of your state plays a major role in how you structure your reducer logic.

A Redux state usually has a plain Javascript object as the top of the state tree. (It is certainly possible to have another type of data instead, such as a single number, an array, or a specialized data structure, but most libraries assume that the top-level value is a plain object.)  The most common way to organize data within that top-level object is to further divide data into sub-trees, where each top-level key represents some "domain" or "slice" of related data.  For example, a basic Todo app's state might look like:

   *{
      visibilityFilter: 'SHOW_ALL',
      todos: [
        {
          text: 'Consider using Redux',
          completed: true,
        },
        {
          text: 'Keep all state in a single tree',
          completed: false
        }
      ]
    }*

In this example, todos and visibilityFilter are both top-level keys in the state, and each represents a "slice" of data for some particular concept.

Most applications deal with multiple types of data, which can be broadly divided into three categories:

Domain data: data that the application needs to show, use, or modify (such as "all of the Todos retrieved from the server")

App state: data that is specific to the application's behavior (such as "Todo #5 is currently selected", or "there is a request in progress to fetch Todos")

UI state: data that represents how the UI is currently displayed (such as "The EditTodo modal dialog is currently open")

Because the store represents the core of your application, you should define your state shape in terms of your domain data and app state, not your UI component tree.  As an example, a shape of state.leftPane.todoList.todos would be a bad idea, because the idea of "todos" is central to the whole application, not just a single part of the UI. The todos slice should be at the top of the state tree instead.

There will rarely be a 1-to-1 correspondence between your UI tree and your state shape. The exception to that might be if you are explicitly tracking various aspects of UI data in your Redux store as well, but even then the shape of the UI data and the shape of the domain data would likely be different.

A typical app's state shape might look roughly like:

   *{
        domainData1 : {},
        domainData2 : {},
        appState1 : {},
        appState2 : {},
        ui : {
            uiState1 : {},
            uiState2 : {},
        }
    }*

# Splitting Reducer Logic

For any meaningful application, putting all your update logic into a single reducer function is quickly going to become unmaintainable.  While there's no single rule for how long a function should be, it's generally agreed that functions should be relatively short and ideally only do one specific thing.  Because of this, it's good programming practice to take pieces of code that are very long or do many different things, and break them into smaller pieces that are easier to understand.

Since a Redux reducer is just a function, the same concept applies.  You can split some of your reducer logic out into another function, and call that new function from the parent function.

These new functions would typically fall into one of three categories:

- Small utility functions containing some reusable chunk of logic that is needed in multiple places (which may or may not be actually related to the specific business logic)

- Functions for handling a specific update case, which often need parameters other than the typical (state, action) pair

- Functions which handle all updates for a given slice of state.  These functions do generally have the typical (state, action) parameter signature

For clarity, these terms will be used to distinguish between different types of functions and different use cases:

- reducer: any function with the signature (state, action) -> newState (ie, any function that could be used as an argument to Array.prototype.reduce)

- root reducer: the reducer function that is actually passed as the first argument to createStore.  This is the only part of the reducer logic that must have the (state, action) -> newState signature.

- slice reducer: a reducer that is being used to handle updates to one specific slice of the state tree, usually done by passing it to combineReducers

- case function: a function that is being used to handle the update logic for a specific action.  This may actually be a reducer function, or it may require other parameters to do its work properly.

- higher-order reducer: a function that takes a reducer function as an argument, and/or returns a new reducer function as a result (such as combineReducers, or redux-undo)

The term "sub-reducer" has also been used in various discussions to mean any function that is not the root reducer, although the term is not very precise.  Some people may also refer to some functions as "business logic" (functions that relate to application-specific behavior) or "utility functions" (generic functions that are not application-specific).

Breaking down a complex process into smaller, more understandable parts is usually described with the term functional decomposition.  This term and concept can be applied generically to any code.  However, in Redux it is very common to structure reducer logic using approach #3, where update logic is delegated to other functions based on slice of state.  Redux refers to this concept as reducer composition, and it is by far the most widely-used approach to structuring reducer logic.  In fact, it's so common that Redux includes a utility function called combineReducers(), which specifically abstracts the process of delegating work to other reducer functions based on slices of state. However, it's important to note that it is not the only pattern that can be used.  In fact, it's entirely possible to use all three approaches for splitting up logic into functions, and usually a good idea as well.

# Store

In the previous sections, we defined the actions that represent the facts about “what happened” and the reducers that update the state according to those actions.

The Store is the object that brings them together. The store has the following responsibilities:

- Holds application state;
- Allows access to state via getState();
- Allows state to be updated via dispatch(action);
- Registers listeners via subscribe(listener);
- Handles unregistering of listeners via the function returned by subscribe(listener).

It's important to note that you'll only have a single store in a Redux application. When you want to split your data handling logic, you'll use reducer composition instead of many stores.

It's easy to create a store if you have a reducer. In the previous section, we used combineReducers() to combine several reducers into one. We will now import it, and pass it to createStore().

   *import { createStore } from 'redux'
    import todoApp from './reducers'
    const store = createStore(todoApp)*

You may optionally specify the initial state as the second argument to createStore(). This is useful for hydrating the state of the client to match the state of a Redux application running on the server.

   *const store = createStore(todoApp, window.STATE_FROM_SERVER)*

# Data Flow

Redux architecture revolves around a strict unidirectional data flow.

This means that all data in an application follows the same lifecycle pattern, making the logic of your app more predictable and easier to understand. It also encourages data normalization, so that you don't end up with multiple, independent copies of the same data that are unaware of one another.

Redux architecture revolves around a strict unidirectional data flow.

This means that all data in an application follows the same lifecycle pattern, making the logic of your app more predictable and easier to understand. It also encourages data normalization, so that you don't end up with multiple, independent copies of the same data that are unaware of one another.

If you're still not convinced, read Motivation and The Case for Flux for a compelling argument in favor of unidirectional data flow. Although Redux is not exactly Flux, it shares the same key benefits.

The data lifecycle in any Redux app follows these 4 steps:

You call store.dispatch(action).

An action is a plain object describing what happened. For example:

 { type: 'LIKE_ARTICLE', articleId: 42 }
 { type: 'FETCH_USER_SUCCESS', response: { id: 3, name: 'Mary' } }
 { type: 'ADD_TODO', text: 'Read the Redux docs.' }

Think of an action as a very brief snippet of news. “Mary liked article 42.” or “‘Read the Redux docs.' was added to the list of todos.”

You can call store.dispatch(action) from anywhere in your app, including components and XHR callbacks, or even at scheduled intervals.

The Redux store calls the reducer function you gave it.

The store will pass two arguments to the reducer: the current state tree and the action. For example, in the todo app, the root reducer might receive something like this:

   *// The current application state (list of todos and chosen filter)
     let previousState = {
       visibleTodoFilter: 'SHOW_ALL',
       todos: [
         {
           text: 'Read the docs.',
           complete: false
         }
       ]
    }
    ​
    // The action being performed (adding a todo)
     let action = {
       type: 'ADD_TODO',
       text: 'Understand the flow.'
    }
    ​
    // Your reducer returns the next application state
    let nextState = todoApp(previousState, action)*

Note that a reducer is a pure function. It only computes the next state. It should be completely predictable: calling it with the same inputs many times should produce the same outputs. It shouldn't perform any side effects like API calls or router transitions. These should happen before an action is dispatched.

The root reducer may combine the output of multiple reducers into a single state tree.

How you structure the root reducer is completely up to you. Redux ships with a combineReducers() helper function, useful for “splitting” the root reducer into separate functions that each manage one branch of the state tree.

Here's how combineReducers() works. Let's say you have two reducers, one for a list of todos, and another for the currently selected filter setting:

   *function todos(state = [], action) {
       // Somehow calculate it...
       return nextState
    }
    ​
    function visibleTodoFilter(state = 'SHOW_ALL', action) {
       // Somehow calculate it...
       return nextState
    }
    ​
    let todoApp = combineReducers({
       todos,
       visibleTodoFilter
    })*

When you emit an action, todoApp returned by combineReducers will call both reducers:

 *let nextTodos = todos(state.todos, action)
  let nextVisibleTodoFilter = visibleTodoFilter(state.visibleTodoFilter, action)*

It will then combine both sets of results into a single state tree:

   *return {
     todos: nextTodos,
     visibleTodoFilter: nextVisibleTodoFilter
   }*

While combineReducers() is a handy helper utility, you don't have to use it; feel free to write your own root reducer!

The Redux store saves the complete state tree returned by the root reducer.

This new tree is now the next state of your app! Every listener registered with store.subscribe(listener) will now be invoked; listeners may call store.getState() to get the current state.

Now, the UI can be updated to reflect the new state. If you use bindings like React Redux, this is the point at which component.setState(newState) is called.

# Usage with React

From the very beginning, we need to stress that Redux has no relation to React. You can write Redux apps with React, Angular, Ember, jQuery, or vanilla JavaScript.

That said, Redux works especially well with libraries like React and Deku because they let you describe UI as a function of state, and Redux emits state updates in response to actions.

We will use React to build our simple todo app.

# Presentational and Container Components

React bindings for Redux embrace the idea of separating presentational and container components.

Most of the components we'll write will be presentational, but we'll need to generate a few container components to connect them to the Redux store. This and the design brief below do not imply container components must be near the top of the component tree. If a container component becomes too complex (i.e. it has heavily nested presentational components with countless callbacks being passed down), introduce another container within the component tree.

Technically you could write the container components by hand using store.subscribe(). We don't advise you to do this because React Redux makes many performance optimizations that are hard to do by hand. For this reason, rather than write container components, we will generate them using the connect() function provided by React Redux, as you will see below.

# Designing Component Hierarchy

Remember how we designed the shape of the root state object? It's time we design the UI hierarchy to match it. This is not a Redux-specific task. Thinking in React is a great tutorial that explains the process (See ReadME in Repository).

Our design brief is simple. We want to show a list of todo items. On click, a todo item is crossed out as completed. We want to show a field where the user may add a new todo. In the footer, we want to show a toggle to show all, only completed, or only active todos.

# Designing Presentational Components

I see the following presentational components and their props emerge from this brief:

TodoList is a list showing visible todos.

- todos: Array is an array of todo items with { id, text, completed } shape.
- onTodoClick(id: number) is a callback to invoke when a todo is clicked.

Todo is a single todo item.

- text: string is the text to show.
- completed: boolean is whether the todo should appear crossed out.
- onClick() is a callback to invoke when the todo is clicked.

Link is a link with a callback.

- onClick() is a callback to invoke when the link is clicked.

Footer is where we let the user change currently visible todos.
App is the root component that renders everything else.

They describe the look but don't know where the data comes from, or how to change it. They only render what's given to them. If you migrate from Redux to something else, you'll be able to keep all these components exactly the same. They have no dependency on Redux.

# Designing Container Components

We will also need some container components to connect the presentational components to Redux. For example, the presentational TodoList component needs a container like VisibleTodoList that subscribes to the Redux store and knows how to apply the current visibility filter. To change the visibility filter, we will provide a FilterLink container component that renders a Link that dispatches an appropriate action on click:

- VisibleTodoList filters the todos according to the current visibility filter and renders a TodoList.

- FilterLink gets the current visibility filter and renders a Link.
- filter: string is the visibility filter it represents.

# Designing Other Components

Sometimes it's hard to tell if some component should be a presentational component or a container. For example, sometimes form and function are really coupled together, such as in the case of this tiny component:

- AddTodo is an input field with an “Add” button

Technically we could split it into two components but it might be too early at this stage. It's fine to mix presentation and logic in a component that is very small. As it grows, it will be more obvious how to split it, so we'll leave it mixed.

# Implementing Components

Let's write the components! We begin with the presentational components so we don't need to think about binding to Redux yet.

# Implementing Presentational Components

These are all normal React components, so we won't examine them in detail. We write functional stateless components unless we need to use local state or the lifecycle methods. This doesn't mean that presentational components have to be functions—it's just easier to define them this way. If and when you need to add local state, lifecycle methods, or performance optimizations, you can convert them to classes.


# Implementing Container Components

Now it's time to hook up those presentational components to Redux by creating some containers. Technically, a container component is just a React component that uses store.subscribe() to read a part of the Redux state tree and supply props to a presentational component it renders. You could write a container component by hand, but we suggest instead generating container components with the React Redux library's connect() function, which provides many useful optimizations to prevent unnecessary re-renders. (One result of this is that you shouldn't have to worry about the React performance suggestion of implementing shouldComponentUpdate yourself.)

To use connect(), you need to define a special function called mapStateToProps that tells how to transform the current Redux store state into the props you want to pass to a presentational component you are wrapping. For example, VisibleTodoList needs to calculate todos to pass to the TodoList, so we define a function that filters the state.todos according to the state.visibilityFilter, and use it in its mapStateToProps:

    *const getVisibleTodos = (todos, filter) => {
      switch (filter) {
        case 'SHOW_COMPLETED':
          return todos.filter(t => t.completed)
        case 'SHOW_ACTIVE':
          return todos.filter(t => !t.completed)
        case 'SHOW_ALL':
        default:
          return todos
        }     
      }
​
      const mapStateToProps = state => {
        return {
          todos: getVisibleTodos(state.todos, state.visibilityFilter)
        }
      }*

In addition to reading the state, container components can dispatch actions. In a similar fashion, you can define a function called mapDispatchToProps() that receives the dispatch() method and returns callback props that you want to inject into the presentational component. For example, we want the VisibleTodoList to inject a prop called onTodoClick into the TodoList component, and we want onTodoClick to dispatch a TOGGLE_TODO action:

   *const mapDispatchToProps = dispatch => {
      return {
        onTodoClick: id => {
          dispatch(toggleTodo(id))
        }
      }
    }*

Finally, we create the VisibleTodoList by calling connect() and passing these two functions:

   *import { connect } from 'react-redux'
    ​
    const VisibleTodoList = connect(
      mapStateToProps,
      mapDispatchToProps
    )(TodoList)
    ​
    export default VisibleTodoList;*

These are the basics of the React Redux API, but there are a few shortcuts and power options so we encourage you to check out its documentation in detail. In case you are worried about mapStateToProps creating new objects too often, you might want to learn about computing derived data with reselect.

# Implementing Other Components

Recall as mentioned previously, both the presentation and logic for the AddTodo component are mixed into a single definition.


# Passing the Store

All container components need access to the Redux store so they can subscribe to it. One option would be to pass it as a prop to every container component. However it gets tedious, as you have to wire store even through presentational components just because they happen to render a container deep in the component tree.

The option we recommend is to use a special React Redux component called <Provider> to magically make the store available to all container components in the application without passing it explicitly. You only need to use it once when you render the root component:
