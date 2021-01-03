---
title: 开始使用React Hooks
date: "2019-03-12"
tags: ["React", "Hook"]
---
**Hooks** 是React16.8最新引入的特性。使用**Hooks**可以在函数式组件中管理state。

**Hooks** 带来的好处可以让更简洁的让UI与状态分离，使代码更加清晰。

## 说明

使用**Hooks**的前提是在函数式组件中。所以不能再使用React类组件的几个生命周期函数(需要通过`useEffect`来实现)

## 开始

前端项目万物基于TODO APP 😂 , 接一下使用**Hooks**来创建一个TODO APP。

### useState

**useState** 接受一个初始化参数， 返回一个值和set方法

todo 组件：

```javascript
const todo = props => {
  const [todoName, setTodoName] = useState('')
  const [todoList, setTodoList] = useState([])
  const inputChangeHandler = event => {
      setTodoName(event.target.value)
  }
  const todoAddHandler = () => {
      setTodoList(todoList.concat(todoName))
  }
  return (
      <React.Fragment>
      <input type="text" placeholder="Todo"
      	value={todoName}
        onChange={inputChangeHandler} />
      <button type="button" onClick={todoAddHandler}>Add</button>
      <ul>
      	{todoList.map((item, index) => (
            <li key={index}>item</li>
        ))}
      </ul>
      </React.Fragment>
  )
}
```

### useEffect

通过向**useEffect**穿不同的参数，可以模拟原来类组件中的生命周期方法。

```javascript
  const mouseMoveHandler = event => {
      console.log(event.clientX, event.clientY)
  }

  useEffect(() => {
    document.addEventListener('mousemove', mouseMoveHandler)
    return () => {
      document.removeEventListener('mousemove', mouseMoveHandler)
    }
  }, [])
```

使用`useEffect(fn, [])`可以差不多的模拟`componentDidMount`。 不传第二个**deps**参数， 每次render都会执行**useEffect**

`useEffect`返回callback 表示 clean up,模拟 `unmount`。在`[]`可以中指定监听的对象**scope** ，当对象改变时，执行`useEffect`。

[这篇文章](https://overreacted.io/a-complete-guide-to-useeffect/)详细的说明了**useEffect**中异步state状态更新和this指向的问题

### useContext

在类组件中使用**React context**,需要使用 jsx

```javascript
<MyContext.Consumer>
  {value => /* render something based on the context value */}
</MyContext.Consumer>
```

这样包起来，然后有了**useContext**, 在子组件直接使用即可

```javascript
const context = useContext(MyContext)
```

### useRef

**useRef**返回一个可变的ref引用。可以把TODO input输入改写为：

```javascript
 const todo = props => {
   const todoInputRef = useRef()
   ...
   //获取input的值
   // const todoName = todoInputRef.current.value
   return (
      <React.Fragment>
      <input type="text" placeholder="Todo"
        ref={todoInputRef}
        ...
      </React.Fragment>
  )
 }
```

### useReducer

在API处理异步的时候当网络慢的时候state会更新错误，比如：

```javascript
  const todoAddHandler = () => {
      axios.post('api', {name: todoName})
      .then(res => {
        setTimeout(() => {
          const todoItem = {id: res.data.name, name: todoName }
          //当连续添加时，当前的todoList只能更新一个值
          setTodoList(todoList.concat(todoItem))
        }, 3000)
      }).catch(err => {
        console.log(err)
      })
  }
```

可以添加一个flagState来处理

```javascript
  const [submittedTodo, setSubmittedTodo] = useState(null)
  useEffect(() => {
      if (submittedTodo) {
          setTodoList(todoList.concat(submittedTodo))
      }
  }, [submittedTodo])
  const todoAddHandler = () => {
      axios.post('api', {name: todoName})
          .then(res => {
          setTimeout(() => {
              const todoItem = {id: res.data.name, name: todoName }
              // 只更新submittedTodo
              setSubmittedTodo(todoItem)
              }, 3000)
      }).catch(err => {
          console.log(err)
      })
  }
```

当然 更好的方法可以使用**useReducer**来解决这个问题

```javascript
const todoListReducer = (state, action) => {
    switch (action.type) {
        case 'ADD':
            return state.concat(action.payload)
        case 'SET':
            return action.payload
        case 'REMOVE':
            return state.filter(todo => todo.id !== action.payload)
        default:
            return state
    }
}

const [todoList, dispatch] = useReducer(todoListReducer, [])
const todoAddHandler = () => {
    axios.post('api', {name: todoName})
        .then(res => {
        setTimeout(() => {
            const todoItem = {id: res.data.name, name: todoName }
            dispatch({type: 'ADD', payload: todoItem})
        }, 3000)
    }).catch(err => {
        console.log(err)
    })
}
```

### useMemo

useMemo 用于优化性能，省略不必要的重新render

比如 将 TODO App 每一个item 封装成一个**List**组件：

```javascript
// List
const list = props => {
  console.log('Rendering the list...')
  return <ul>
  {
      props.items.map(todo => (
        <li key={todo.id} onClick={props.onClick.bind(this,todo.id)}>{todo.name}</li>
      ))
  }
  </ul>
}
// TODO
const todo = props => {
    ...
    return (
     ....
        {useMemo(() =>
        <List items={todoList} onClick={todoRemoveHandler}/>, [todoList]
      )}
    )
}
```



### 自定义hook

自定义一个**useFormInput**

```javascript
// form.js
export const useFormInput = () => {
  const [value, setValue] = useState('')
  const [validity, setValidity] = useState(false)

  const inputChangeHandler = event => {
    setValue(event.target.value)
    if(event.target.value.trim() === '') {
      setValidity(false)
    } else {
      setValidity(true)
    }
  }

  return {
    value: value,
    onChange: inputChangeHandler,
    validity
  }
}
//todo.js
const todo = props => {
    const todoInput = useFormInput()
    return (
        ...
        <React.Fragment>
      <input type="text" placeholder="Todo"
      onChange={todoInput.onChange}
      value={todoInput.value}
      style={{backgroudColor: todoInput.validity ? 'transparent' : 'red'}}
       />
		...
    )
}
```



[样例](https://codesandbox.io/s/8ylx565y0)

## 参考

- [A Complete Guide to useEffect](https://overreacted.io/a-complete-guide-to-useeffect/)
- [How to fetch data with React Hooks?](https://www.robinwieruch.de/react-hooks-fetch-data/)
