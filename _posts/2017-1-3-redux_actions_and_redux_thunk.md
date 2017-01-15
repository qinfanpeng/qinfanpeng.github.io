---
layout: post
title:  "Redux Actions & Redux Thunk"
date:   2017-01-03 12:55:19 +0800
categories: jekyll update
---

[slide]

# Redux Actions & Redux Thunk
## qinfanpeng


[slide]

## Agenda
* FSA
* Action Creator
* redux-actions
* redux-promise
* redux-thunk


[slide]

## Agenda
* **FSA**
* Action Creator
* **redux-actions**
* redux-promise
* **redux-thunk**


[slide]

## Rails REST Routes

---
HTTP Verb | Path | Controller#Action | Used for
:-------|:------:|-------:|--------
GET | /todos | todos#index | display a list of all todos
GET | /todos/new | todos#new | return an HTML form for creating a new todo
POST | /todos | todos#create | create a new todo
GET | /todos/:id | todos#show | display a specific todo
GET | /todos/:id/edit | todos#edit | return an HTML form for editing a todo
PATCH/PUT | /todos/:id | todos#update | update a specific todo
DELETE | /todos/:id | todos#destroy | delete a specific todo

[note]
Restul中由Http Verb 和 Path 共同决定 “要做什么”
[/note]

[slide]

## Rails Controller & Actions

```ruby
class TodosController < ApplicationController
  # GET /todos
  def index
    @todos = Todo.all
  end
  
  # GET /todos/:id
  def show
    @todo = Todo.find(params[:id])
  end
  
  # GET /todos/new
  def new
    @todo = Todo.new
  end

  # POST /todos
  def create
    @todo = Todo.new(params[:todo])
 
    if @todo.save
      redirect_to @todo
    else
      render 'new'
    end  
  end
  
  # ...
end 

```

[note]
Controller 中的 Action 可以从 params中获取 http 参数。
结合起来看，Flux 中的 Action 要做的事情也极其类似：
1. 决定“要做什么”
2. 携带必要的参数
[/note]

[slide]

## Describe Redux Action in One Sentence
* What to do ? {:&.fadeIn}
* And along with necessary data


[slide]

## FSA(Flux Standard Action)
```javascript
{
  type: ADD_TODO,
  payload: {
    text: 'Do something.'  
  }
}
```

```javascript
{
  type: ADD_TODO,
  payload: new Error(),
  error: true
}
```


[slide]

## Flux Standard Action Specifications

* An action **MUST** {:&.fadeIn}
  * be a plain JavaScript object. {:&.fadeIn}
  * have a `type` property.

* An action **MAY** have:
  * an `error` property. {:&.fadeIn}
  * a `payload` property.
  * a `meta` property.

* An action **MUST NOT** include properties other than `type`, `payload`, `error`, and `meta`.  


[slide]

## Benefit of using FSA

* Standard to Follow {:&.fadeIn}
* Easy to switch Flux implementations
* More friendly to middleware


[slide]

## Use Action Literal Directly

* {:&.fadeIn}

```javascript
store.dispatch({
  type: ADD_TODO,
  payload: {
    text: 'Do one thing.'  
  }
})
```

```javascript
store.dispatch({
  type: ADD_TODO,
  payload: {
    text: 'Do another thing.'  
  }
})

```

[note]
直接使用 Action 字面量虽然直接，但繁琐、重复、极像样板（Boilerplate）、代码将代码的主要目的淹没在了细节中（暴露了不必要的细节）
[/note]

[slide]

## Encapsulate Action in Action Creator

* {:&.fadeIn}

```javascript
const addTodo = (text) => {
  return {
    type: ADD_TODO,
    payload: { text }
  }  
}
```

```javascript
store.dispatch(addTodo('Do one thing.'))
store.dispatch(addTodo('Do another thing.'))
```

[note]
利用 Action Creator 封装了细节，减少重复，且令调用代码的意图一目了然。
[/note]


[slide]

## Todo Actions (by Action Creators)

* {:&.fadeIn}

```javascript
const addTodo = (text) => {
  return {
    type: ADD_TODO,
    payload: { id: uuid.v4(), text }
  }
}

```

```javascript
const updateTodo = (id, text) => {
  return {
    type: UPDATE_TODO,
    payload: { id, text }
  }
}
```

```javascript
const clearTodos = () => {
  return {type: CLEAR_TODOS}
}

```

[note]
即使采用了 Action Creator，但还是有重复的影子，不够完美。
[/note]


[slide]

## Use redux-actions to Simplify Action Creator
* {:&.fadeIn}

```javascript
// Solution One, which is perfer
const addTodo = (text) => {
  return createAction(ADD_TODO)({ id: uuid.v4(), text })
}
store.dispatch(addTodo(1, 'Do something'))

// Solution Two
const addTodo = createAction(ADD_TODO)
store.dispatch(addTodo({id: uuid.v4(), text: 'Do something'}))
```

```javascript
const updateTodo = (id, text) => {
  return createAction(UPDATE_TODO)({ id, text })
}
```

```javascript
const clearTodos = createAction(CLEAR_TODOS_TODO)
store.dispatch(clearTodos())
```

[note]
借助 redux-actions 的 createAction 进一步消除重复，简化代码。
[/note]


[slide]

## Common Misuse of redux-actions

* {:&.fadeIn}

```javascript
const deleteTodo = (id) => {
  return createAction(DELETE_TODO)(id)
}
```

```javascript
// Same as
const deleteTodo = (id) => {
  return {
    type: DELETE_TODO,
    payload: id
  }
}
```

```javascript
const todos = handleActions({
  [DELETE_TODO]: (state, { payload }) => {
    // What the hell is payload ?
    return deleteTodo(state, payload)
  },
}, [])

```

[note]
payload 中的数据应该有自己的结构，而非直接塞给 payload，否则的话，reducer 里代码根本不知道 payload 里有啥。
[/note]


[slide]

## the Way to use redux-actions

* {:&.fadeIn}

```javascript
const deleteTodo = (id) => {
  return createAction('DELETE_TODO')({ id })
}
```

```javascript
const todos = handleActions({
  DELETE_TODO: (state, { payload }) => {
    return deleteTodo(state, payload.id)
  },
}, [])

```

[slide]

## use redux-promise to Call API

```javascript
const fetchTodo = (id) => {
  return {
    type: 'FETCH_TODO',
    payload: fetch(`/todos/${id}`)
  }
}

// Or 
const fetchTodo = (id) => {
  return fetch(`/todos/${id}`)
           .then(todo => {
  			   return {
               type: 'FETCH_TODO',
               payload: todo
             }         
           })
}

store.dispatch(fetchTodo(1))
```

[note]
redux-promise 这个 middleweare 可以识别 payload 为 promise 的 action 或 全由 promise 组成的整个 action。
[/note]


[slide]

[magic data-transition="cover-circle"]
## use redux-thunk to Call API

```javascript
const fetchTodo = (id) => {
  return (dispatch, getState) => {
    fetch(`/todos/${id}`)
      .then(todo => {
        dispatch(createAction(FETCH_TODO)({ todo }))
      })
      .catch(error => {
        dispatch(createAction(FETCH_TODO)(error))
      })
  }
}

store.dispatch(fetchTodo(1))

```
[note]
redux-thunk 也可处理 promise。
[/note]

=====
## use redux-thunk to Conditional Dispatch

```javascript
const addTodo = (text) => {
  return (dispatch, getState) => {
    if (getState().todos.length <= 3) {
    	dispatch(createAction(FETCH_TODO)({ todo }))
    }
  }
}

```

[note]
redux-thunk 还可用于按需调用。
[/note]


=====
## Use redux-thunk to Dispatch Multiple Actions at Once

```javascript
const fetchTodo = (id) => {
  return (dispatch, getState) => {
    dispatch(createAction(START_FETCH_TODO)({ todo }))
    
    fetch(`/todos/${id}`)
      .then(todo => {
        dispatch(createAction(FETCH_TODO_SUCCESS)({ todo }))
        // dispatch(createAction(TODO_RECIVED)({ todo }))
      })
      .catch(error => {
        dispatch(createAction(FETCH_TODO_FAIL)(error))
      })
  }
}

```

[note]
redux-thunk 还可用于批量调用多个 action。显然 redux-thunk 的适用场景要比 redux-promise 要广。
[/note]


[/magic]


[slide]

[magic data-transition="cover-circle"]
## Overuse redux-thunk

```javascript
const updateTodo = (id, text) => {
  return (dispatch) => {
  	 return dispatch(createAction(UPDATE_TODO)({ id, text }))
  }
}

```

[note]
并非所有的 action 都要写成 thunk 形式。
[/note]


====
## Overuse redux-thunk

```javascript
const mapStateToProps = (state, ownProps) => ({})
const mapDispatchToProps = (dispatch, ownProps) => ({
  updateTodo: (id, text) => {
    return (dispatch) => {
      return dispatch(createAction(UPDATE_TODO)({ id, text }))
    }
  },
  addTodo: (text) => {
    return (dispatch) => {
      return dispatch(createAction(ADD_TODO)({ id: uuid.v4(), text }))
    }
  },
  toggleTodo: (id) => {
    return (dispatch) => {
      return dispatch(createAction(TOGGLE_TODO)({ id: uuid.v4(), text }))
    }  
  },                
})

@connect(mapStateToProps, mapDispatchToProps)
```

[note]
把上面的方法 inline 后，就显得更碍眼了。
[/note]


====
## Overuse reason ？
* Not familiar with `bindActionCreators`
* Suppose will take over everything 
* Copy & paste

====

## react-redux will auto use `bindActionCreators`

```javascript
const mapStateToProps = (state, ownProps) => ({})
const mapDispatchToProps = (dispatch, ownProps) => ({
  updateTodo: (id, text) => {
    return (dispatch) => {
      return dispatch(createAction(UPDATE_TODO)({ id, text }))
    }
  },
  addTodo: (text) => {
    return (dispatch) => {
      return dispatch(createAction(ADD_TODO)({ id: uuid.v4(), text }))
    }
  },
  toggleTodo: (id) => {
    return (dispatch) => {
      return dispatch(createAction(TOGGLE_TODO)({ id: uuid.v4(), text }))
    }  
  },                
})
```

* ↓

```javascript
const mapStateToProps = (state, ownProps) => ({})
const mapDispatchToProps = {
  updateTodo: (id, text) => createAction(UPDATE_TODO)({ id, text }),
  addTodo: (text) => createAction(ADD_TODO)({ id: uuid.v4(), text }),
  toggleTodo: (id) => createAction(TOGGLE_TODO)({ id: uuid.v4(), text }),                
}

// will auto call bindActionCreators(mapDispatchToProps, dispatch)

@connect(mapStateToProps, mapDispatchToProps)
```

====

## redux-thunk Source Code

```javascript
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => next => action => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }

    return next(action);
  };
}

const thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;

export default thunk;
```

====
## Fix The Overuse of redux-thunk

```javascript
const updateTodo = (id, text) => {
  return (dispatch) => {
  	 return createAction('UPDATE_TODO')({ id, text })
  }
}
```

* ↓

```javascript
const updateTodo = (id, text) => createAction('UPDATE_TODO')({ id, text })
```

[/magic]


[slide]

## Thank You & QA
