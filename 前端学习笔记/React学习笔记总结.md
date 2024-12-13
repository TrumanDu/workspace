# React 学习笔记总结

## props vs state

在 React 中有两种“模型”数据：props 和 state。下面是它们的不同之处:

-   props 像是你传递的参数 至函数。它们使父组件可以传递数据给子组件，定制它们的展示。举个例子，Form 可以传递 color prop 至 Button。
-   state 像是组件的内存。它使组件可以对一些信息保持追踪，并根据交互来改变。举个例子，Button 可以保持对 isHovered state 的追踪。

### 响应事件

传递给事件处理函数的函数应直接传递，而非调用。例如：

| 传递一个函数（正确）                    | 调用一个函数（错误）               |
| :-------------------------------------- | :--------------------------------- |
| `<button onClick={handleClick}>`        | `<button onClick={handleClick()}>` |
| `<button onClick={() => alert('...')}>` | `<button onClick={alert('...')}>`  |

## Hooks

### useState 示例：管理状态

使用 useState 处理函数组件的状态，例如：

```
import React, { useState } from 'react';

  function Example() {
    const [count, setCount] = useState(0);

    return (
      <div>
        <p>You clicked {count} times</p>
       <button onClick={() => setCount(count + 1)}>
        Click me
       </button>
     </div>
   );
 }
```

这里要注意的是 setCount 异步函数，它不会改变 count,而是创建一个新的 count。而且在接受同一个 state 对象时，即使其对象里的属性变了，但对象地址没变，是不会更新视图的。

函数中的 setXXX 注意事项：

1. 不可局部更新视图

2. 地址一定要变

### useEffect 示例：处理副作用

`useEffect` 用于在函数组件中处理**副作用**，比如数据获取、订阅、或手动更改 DOM。

```
import React, { useState, useEffect } from 'react';

function DynamicTitle() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    // 修改页面标题
    document.title = `You clicked ${count} times`;
  }, [count]); // 依赖 count 变化

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>Click me</button>
    </div>
  );
}
```

第一次渲染运行：

```
useEffect(()=>{
      console.log("第一次渲染");
},[])

```

只有 n 变化才会调用

```

useEffect(()=>{
      console.log("n 变了");
},[n])

```

任意值变化，都会调用

```
useEffect(()=>{
      console.log("任意属性变了");
})
```

模拟 componentWillUnmount

```
useEffect(()=>{
      console.log("任意属性变了");
      return ()=>{
            console.log("该组件要销毁了")
      }
})
```

### useContext 示例：共享状态

`useContext` 用于在组件树中共享数据，无需逐层传递 props。

```
import React, { createContext, useContext } from 'react';

// 创建 Context
const ThemeContext = createContext('light');

function ThemeButton() {
  const theme = useContext(ThemeContext); // 使用 useContext 获取主题
  return <button style={{ background: theme === 'dark' ? '#333' : '#fff' }}>I am {theme} theme</button>;
}

function App() {
  return (
    <ThemeContext.Provider value="dark">
      <ThemeButton />
    </ThemeContext.Provider>
  );
}

export default App;
```

### useRef 示例：访问 DOM 元素或保存可变值

`useRef` 可以访问 DOM 节点或保存一个在组件生命周期中不会重新渲染的变量。

```
import React, { useRef } from 'react';

function FocusInput() {
  const inputRef = useRef(null);

  const focusInput = () => {
    inputRef.current.focus(); // 聚焦输入框
  };

  return (
    <div>
      <input ref={inputRef} type="text" />
      <button onClick={focusInput}>Focus Input</button>
    </div>
  );
}

export default FocusInput;
```

## 路由

### Hooks 路由

-   useHistory
-   useLocation
-   useParams
-   useRouteMatch

```
import { useHistory } from "react-router-dom";

function HomeButton() {
  let history = useHistory();

  function handleClick() {
    history.push("/home");
  }

  return (
    <button type="button" onClick={handleClick}>
      Go home
    </button>
  );
}
```

useLocation: 获取当前的路由信息

```
import { useLocation } from 'react-router-dom';
function ShowLocation() {
  const location = useLocation(); // 获取 location 对象

  return (
    <div>
      <h2>Current URL</h2>
      <p>Pathname: {location.pathname}</p>
      <p>Search: {location.search}</p>
      <p>Hash: {location.hash}</p>
    </div>
  );
}
```

useParams: 获取动态路由参数

```
import { useParams } from 'react-router-dom';
function UserProfile() {
  const { userId } = useParams(); // 获取动态参数 userId

  return (
    <div>
      <h2>User Profile</h2>
      <p>User ID: {userId}</p>
    </div>
  );
}
```

useRouteMatch: 匹配当前路由

```
import { useRouteMatch } from 'react-router-dom';
function Dashboard() {
  const match = useRouteMatch('/dashboard'); // 检查是否匹配 /dashboard

  return (
    <div>
      <h2>Dashboard</h2>
      {match && <p>Welcome to the Dashboard!</p>}
    </div>
  );
}

```
