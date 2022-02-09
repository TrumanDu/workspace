# 春雷行动-前端技术之CSS必备知识

## 前言

这篇文章不是教程，也不是让读者能够学会和掌握CSS,而是我的学习笔记。文章的目的是让我实践，系统的了解一下一些必备知识，仅此而已。

如果你需要这方面的知识详细学习的话，推荐你看参考的链接。内容全部来源于此。

## 正文

### 五种经典布局

#### 1.空间居中

```css
.parent{
            display: grid;
            place-items: center;
        }
```

核心代码是`place-items`属性,那个是它的简写形式`place-items: <align-items> <justify-items>;` 两者相同的话可以省略。

左上角 `place-items: start;`

右下角`place-items: end;`

#### 2.并列式布局

并列式布局就是多个项目并列。容器设置成 Flex 布局，内容居中（`justify-content`）可换行（`flex-wrap`）

```css
.container{
            display: flex;
            justify-content: center;
            flex-wrap: wrap;
        }
```

项目上面只用一行`flex`

```css
.col{
       flex: 1 1 300px;
     }
```

`flex`属性是`flex-grow`、`flex-shrink`、`flex-basis`这三个属性的简写形式 `flex: <flex-grow> <flex-shrink> <flex-basis>;`

- `flex-basis`：项目的初始宽度。
- `flex-grow`：指定如果有多余宽度，项目是否可以扩大。
- `flex-shrink`：指定如果宽度不足，项目是否可以缩小。

`flex: 0 1 300px;` 表示项目初始宽度是300，宽度不够的话可以缩小，但是不可以扩大。`flex: 1 1 300px;`，表示项目始终会占满所有宽度。

#### 3.两栏式布局

```
.container{
            display: grid;
            grid-template-columns: minmax(100px,20%) 1fr;
        }
```

`grid-template-columns`指定页面分成两列。第一列的宽度是`minmax(150px, 25%)`，即最小宽度为`150px`，最大宽度为总宽度的25%；第二列为`1fr`，即所有剩余宽度。切记div作为容器，需要项目（div）。

#### 4.三明治布局

三明治布局指的是，页面在垂直方向上，分成三部分：页眉、内容区、页脚。

```
.container{
            display: grid;
            grid-template-rows: auto 1fr auto;
        }
```

grid-template-rows是指垂直方向。head和foot都是跟随内容自动变化，中间区域为剩余大小

#### 5.圣杯布局

圣杯布局是最常用的布局，所以被比喻为圣杯。它将页面分成五个部分，除了页眉和页脚，内容区分成左边栏、主栏、右边栏。

```
  body{
            display: grid;
            grid-template: auto 1fr auto/auto 1fr auto;
        }
        header{
            grid-column: 1 / 4;
        }
        .main{
            grid-column: 2 / 3;
        }
        .left{
            grid-column: 1 / 2;
        }
        .right{
            grid-column: 3 / 4;
        }
        footer{
            grid-column: 1 / 4;
        }
```

grid-template是`grid-template-rows`和`grid-template-columns`的简写，`grid-template: <grid-template-rows> / <grid-template-columns>`

grid-column是`grid-column-start`和`grid-column-end`的简写，`grid-column: <start-line> / <end-line>`表示垂直第几根线。上面代码的`grid-column: 1 / 4`代表从第1列到第4列。

### CSS使用技巧

#### 高度100%

有的时候，我们发现高度设置为100%不生效，这个是因为百分比相对于父元素，因为我们将所有的父元素都设置为`height:100%`

```html
    <style>
        html,body{
            height: 100%;
            margin: 0px; 
        }
        .parent{
            background-color: lightskyblue;
            width: 100%;
            height: 100%;
        }
    </style>
```



```
<div class="parent"></div>
```

当然使用`100vh`就没有上面的问题。

#### **文字的水平居中**

```
text-align:center;
```

#### **容器的水平居中**

```
div#container {
　　　　width:760px;
　　　　margin:0 auto;
　　}
```

先为该容器设置一个明确宽度，然后将margin的水平值设为auto即可。

#### **文字的垂直居中**

```
height: 35px; line-height: 35px;
```

如果有n行文字，那么将行高设为容器高度的n分之一即可。

#### **容器的垂直居中**

```
<div id="big">
　　　　<div id="small">
　　　　</div>
</div>

div#big{
　　　　position:relative;
　　　　height:480px;
　　}
　div#small {
　　　　position: absolute;
　　　　top: 50%;
　　　　height: 240px;
　　　　margin-top: -120px;
　　}　
```

将大容器的定位为relative。将小容器定位为absolute，再将它的左上角沿y轴下移50%，最后将它margin-top上移本身高度的50%即可。

#### 容器水平垂直居中

```
.container{
            position: relative; 
            background-color: yellow;
            height: 100vh;
            width: 100vw;
        }
        .col{
            background-color: red;
            position:absolute;
            top:50%;
            left: 50%;
            height: 200px;
            width: 200px;
            margin: -100px 0 0 -100px;
        }
```

margin: top right bottom left

#### **图片宽度的自适应**

```
img {max-width: 100%}
```

#### **3D按钮**

```
div#button {
　　　　background: #888;
　　　　border: 1px solid;
　　　　border-color: #999 #777 #777 #999;
　　}
```

#### **CSS的优先性**

行内样式 > id样式 > class样式 > 标签名样式

#### **禁止文字自动换行**

```
white-space:nowrap;
```



## 参考

1. [CSS使用技巧](http://www.ruanyifeng.com/blog/2010/03/css_cookbook.html)

2. [CSS 定位详解](http://www.ruanyifeng.com/blog/2019/11/css-position.html)

3. [只要一行代码，实现五种 CSS 经典布局](https://www.ruanyifeng.com/blog/2020/08/five-css-layouts-in-one-line.html)

4. [CSS Grid 网格布局教程](http://www.ruanyifeng.com/blog/2019/03/grid-layout-tutorial.html)

5. [Flex 布局教程：语法篇](http://www.ruanyifeng.com/blog/2015/07/flex-grammar.html)

6. [CSS实现居中的方式](https://www.cnblogs.com/ghq120/p/10939835.html)