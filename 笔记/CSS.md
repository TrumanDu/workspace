# CSS

## CSS 选择器

### 标签选择器

标签选择器的选择符是html 元素

```
p{
 color：red;
}
```



### 类选择器

类选择器的选择符是以`.`开头

```
.selfColor{
 color：red;
}
```



### ID选择器

id 选择器的选择符是“#”

```
#myId{
 color：red;
}
```



### 通配符选择器

```
*{
 color：red;
}
```

### 高级选择器

- **后代选择器**： 空格分隔 `.div1 p`

- **交集选择器**： 紧密相连，一般以标签开头 `div.test`   `p#id`

- **并集选择器**： 逗号分隔 

  ```
  p,
  h1,
  #myId
  .myClass{
  color:red;
  }
  ```

- **子代选择器**： 用`>`标识

  ```
  div > p {
  color:red;
  }
  ```

  

- **序选择器**： 

  ```
  ul li:first-child{
  //...
  }
  ul li:last-child{
  //...
  }
  ```

  

- **下一个兄弟选择器**： 用`+`标识，表示紧挨着的第一个兄弟元素

  ```
  div+p {
  color:red;
  }
  ```

  

  ## 伪类

  伪类用`:`来表示，包含静态和动态两种伪类

  ```
  // 静态
  :link 点击之前
  :visited 链接访问过后
  // 动态
  :hover
  :active
  :focus
  ```

  ## 