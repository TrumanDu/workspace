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

### useImmer

`npm install use-immer`

通过使用 Immer，你写出的代码看起来就像是你“打破了规则”而直接修改了对象,useImmer 通过代理和草稿机制**让你像直接修改对象一样操作状态**，同时自动生成新的不可变状态。

```
import { useImmer } from 'use-immer'
  const [person, updatePerson] = useImmer({
    name: 'Niki de Saint Phalle',
    artwork: {
      title: 'Blue Nana',
      city: 'Hamburg',
      image: 'https://i.imgur.com/Sd1AgUOm.jpg',
    }
  });

  function handleNameChange(e) {
    updatePerson(draft => {
      draft.name = e.target.value;
    });
  }
```

### useMemo

useMemo 是一个 React Hook，它在每次重新渲染的时候能够缓存计算的结果。

```
import { useMemo, useState } from 'react';

function TodoList({ todos, filter }) {
  const [newTodo, setNewTodo] = useState('');
  const visibleTodos = useMemo(() => {
    // ✅ 除非 todos 或 filter 发生变化，否则不会重新执行
    return getFilteredTodos(todos, filter);
  }, [todos, filter]);
  // ...
}
```

getFilteredTodos 如果不是一个耗时计算，这里其实不需要使用 useMemo

## 路由

| 功能       | 使用方式                                                   |
| ---------- | ---------------------------------------------------------- |
| 基本导航   | 使用 `<Route>` 定义路由，`<Link>` 创建导航链接。           |
| 动态路由   | 通过 `/path/:param` 定义动态参数，`useParams()` 获取参数。 |
| 嵌套路由   | 结合 `useRouteMatch` 实现路径动态拼接，处理子路由。        |
| 路由重定向 | 使用 `<Redirect>` 或条件渲染实现页面跳转。                 |
| 404 页面   | 使用 `Switch` 和默认路由显示 404 页面。                    |
| 查询参数   | 使用 `useLocation` 和 `URLSearchParams` 解析查询字符串。   |

### 路由学习

#### 简单路由切换

```
import React from 'react';
import { BrowserRouter as Router, Route, Link } from 'react-router-dom';

function Home() {
  return <h2>Home Page</h2>;
}

function About() {
  return <h2>About Page</h2>;
}

function App() {
  return (
    <Router>
      <nav>
        <Link to="/">Home</Link>
        <br />
        <Link to="/about">About</Link>
      </nav>

      <Route exact path="/" component={Home} />
      <Route path="/about" component={About} />
    </Router>
  );
}

export default App;

```

**解释：**

-   <Router>: 包裹整个应用，启用路由功能。
-   <Link>: 创建路由导航的超链接，不会刷新页面。
-   <Route>: 定义路径和对应的组件。exact：确保完全匹配路径。

#### 嵌套导航

```
import React from 'react';
import { BrowserRouter as Router, Routes, Route, Outlet } from 'react-router-dom';

function Layout() {
  return (
    <div>
      <h1>App Layout</h1>
      <Outlet /> {/* 渲染子路由 */}
    </div>
  );
}

function Dashboard() {
  return <h2>Dashboard</h2>;
}

function Settings() {
  return <h2>Settings</h2>;
}

function App() {
  return (
    <Router>
      <Routes>
        <Route path="/" element={<Layout />}>
          <Route path="dashboard" element={<Dashboard />} />
          <Route path="settings" element={<Settings />} />
        </Route>
      </Routes>
    </Router>
  );
}

export default App;


```

**解释**：

-   `<Outlet>` 是嵌套路由的占位符。
-   `Layout` 始终渲染，而 `Dashboard` 和 `Settings` 根据路径匹配。

#### 路由重定向

```
import React from 'react';
import { BrowserRouter as Router, Route, Redirect } from 'react-router-dom';

function Login() {
  const isLoggedIn = false; // 模拟登录状态
  if (isLoggedIn) {
    return <Redirect to="/dashboard" />;
  }
  return <h2>Please log in</h2>;
}

function Dashboard() {
  return <h2>Dashboard</h2>;
}

function App() {
  return (
    <Router>
      <Route path="/login" component={Login} />
      <Route path="/dashboard" component={Dashboard} />
    </Router>
  );
}

export default App;

```

#### 控制路由匹配

在 React Router v6 中，`<Routes>` 是用来替代 v5 中的 `<Switch>`

```
import React from 'react';
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';

function Home() {
  return <h2>Home Page</h2>;
}

function About() {
  return <h2>About Page</h2>;
}

function App() {
  return (
    <Router>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
         <Route path="*" element={<NotFound />} /> {/* 默认匹配 */}
      </Routes>
    </Router>
  );
}

export default App;
```

-   只有第一个匹配的路由（如 `/`）会渲染。
-   **如果不用 `<Routes>`，多个 `<Route>` 都可能渲染（在 v5 或更早版本中会发生）。**

### Hooks 路由

-   useHistory
-   useLocation
-   useParams
-   useRouteMatch
-   useNavigate

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

useNavigate 声明式导航，用于代码中动态跳转路径。

```
const navigate = useNavigate();
navigate('/details/1');
```
