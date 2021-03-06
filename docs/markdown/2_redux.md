# Redux

## Action Creators

### KISS Style
This pattern is focused on a KISS principle - to stay clear of complex proprietary abstractions and follow simple and familiar JavaScript const based types:

Advantages:
- simple "const" based types
- familiar to standard JS usage

Disadvantages:
- significant amount of boilerplate and duplication
- necessary to export both action types and action creators to re-use in other places, e.g. `redux-saga` or `redux-observable`

::example='../../playground/src/redux/counters/actions.ts'::
::usage='../../playground/src/redux/counters/actions.usage.ts'::

[⇧ back to top](#table-of-contents)

### DRY Style
In a DRY approach, we're introducing a simple factory function to automate the creation process of type-safe action creators. The advantage here is that we can reduce boilerplate and repetition significantly. It is also easier to re-use action creators in other layers thanks to `getType` helper function returning "type constant".

Advantages:
- using factory function to automate creation of type-safe action creators
- less boilerplate and code repetition than KISS Style
- getType helper to obtain action creator type (this makes using "type constants" unnecessary)

```ts
import { createAction, getType } from 'react-redux-typescript';

// Action Creators
export const actionCreators = {
  incrementCounter: createAction('INCREMENT_COUNTER'),
  showNotification: createAction('SHOW_NOTIFICATION', 
    (message: string, severity: Severity = 'default') => ({
      type: 'SHOW_NOTIFICATION', payload: { message, severity },
    })
  ),
};

// Usage
store.dispatch(actionCreators.incrementCounter(4)); // Error: Expected 0 arguments, but got 1.
store.dispatch(actionCreators.incrementCounter()); // OK: { type: "INCREMENT_COUNTER" }
getType(actionCreators.incrementCounter) === "INCREMENT_COUNTER" // true

store.dispatch(actionCreators.showNotification()); // Error: Supplied parameters do not match any signature of call target.
store.dispatch(actionCreators.showNotification('Hello!')); // OK: { type: "SHOW_NOTIFICATION", payload: { message: 'Hello!', severity: 'default' } }
store.dispatch(actionCreators.showNotification('Hello!', 'info')); // OK: { type: "SHOW_NOTIFICATION", payload: { message: 'Hello!', severity: 'info' } }
getType(actionCreators.showNotification) === "SHOW_NOTIFICATION" // true
```

[⇧ back to top](#table-of-contents)

---

## Reducers
Relevant TypeScript Docs references:  
- [Discriminated Union types](https://www.typescriptlang.org/docs/handbook/advanced-types.html)
- [Mapped types](https://www.typescriptlang.org/docs/handbook/advanced-types.html) e.g. `Readonly` & `Partial`  

### Tutorial
Declare reducer `State` type definition with readonly modifier for `type level` immutability
```ts
export type State = {
  readonly counter: number,
};
```

Readonly modifier allow initialization, but will not allow rassignment highlighting an error
```ts
export const initialState: State = {
  counter: 0,
}; // OK

initialState.counter = 3; // Error, cannot be mutated
```

#### Caveat: Readonly does not provide recursive immutability on objects
> This means that readonly modifier does not propagate immutability on nested properties of objects or arrays of objects. You'll need to set it explicitly on each nested property.

```ts
export type State = {
  readonly counterContainer: {
    readonly readonlyCounter: number,
    mutableCounter: number,
  }
};

state.counterContainer = { mutableCounter: 1 }; // Error, cannot be mutated
state.counterContainer.readonlyCounter = 1; // Error, cannot be mutated

state.counterContainer.mutableCounter = 1; // No error, can be mutated
```

> There are few utilities to help you achieve nested immutability. e.g. you can do it quite easily by using convenient `Readonly` or `ReadonlyArray` mapped types.

```ts
export type State = Readonly<{
  countersCollection: ReadonlyArray<Readonly<{
    readonlyCounter1: number,
    readonlyCounter2: number,
  }>>,
}>;

state.countersCollection[0] = { readonlyCounter1: 1, readonlyCounter2: 1 }; // Error, cannot be mutated
state.countersCollection[0].readonlyCounter1 = 1; // Error, cannot be mutated
state.countersCollection[0].readonlyCounter2 = 1; // Error, cannot be mutated
```

> _There are some experiments in the community to make a `ReadonlyRecursive` mapped type, but I'll need to investigate if they really works_

[⇧ back to top](#table-of-contents)

### Examples

#### Reducer with classic `const types`

::example='../../playground/src/redux/counters/reducer.ts'::

[⇧ back to top](#table-of-contents)

#### Reducer with getType helper from `react-redux-typescript`
```ts
import { getType } from 'react-redux-typescript';

export const reducer: Reducer<State> = (state = 0, action: RootAction) => {
  switch (action.type) {
    case getType(actionCreators.increment):
      return state + 1;
      
    case getType(actionCreators.decrement):
      return state - 1;

    default: return state;
  }
};
```

[⇧ back to top](#table-of-contents)

---

## Store Configuration

### Create Root State and Root Action Types

#### `RootState` - interface representing redux state tree
Can be imported in connected components to provide type-safety to Redux `connect` function

::example='../../playground/src/redux/root-reducer.ts'::

[⇧ back to top](#table-of-contents)

#### `RootAction` - union type of all action objects
Can be imported in various layers receiving or sending redux actions like: reducers, sagas or redux-observables epics

::example='../../playground/src/redux/root-action.ts'::

[⇧ back to top](#table-of-contents)

### Create Store

When creating the store, use rootReducer. This will set-up a **strongly typed Store instance** with type inference.
> The resulting store instance methods like `getState` or `dispatch` will be type checked and expose type errors

::example='../../playground/src/store.ts'::

[⇧ back to top](#table-of-contents)

---

## Async Flow

### "redux-observable"

```ts
// import rxjs operators somewhere...
import { combineEpics, Epic } from 'redux-observable';

import { RootAction, RootState } from '@src/redux';
import { saveState } from '@src/services/local-storage-service';

const SAVING_DELAY = 1000;

// persist state in local storage every 1s
const saveStateInLocalStorage: Epic<RootAction, RootState> = (action$, store) => action$
  .debounceTime(SAVING_DELAY)
  .do((action: RootAction) => {
    // handle side-effects
    saveState(store.getState());
  })
  .ignoreElements();

export const epics = combineEpics(
  saveStateInLocalStorage,
);
```

[⇧ back to top](#table-of-contents)

---

## Selectors 

### "reselect"

```ts
import { createSelector } from 'reselect';

import { RootState } from '@src/redux';

export const getTodos =
  (state: RootState) => state.todos.todos;

export const getTodosFilter =
  (state: RootState) => state.todos.todosFilter;

export const getFilteredTodos = createSelector(
  getTodos, getTodosFilter,
  (todos, todosFilter) => {
    switch (todosFilter) {
      case 'completed':
        return todos.filter((t) => t.completed);
      case 'active':
        return todos.filter((t) => !t.completed);

      default:
        return todos;
    }
  },
);
```

[⇧ back to top](#table-of-contents)
