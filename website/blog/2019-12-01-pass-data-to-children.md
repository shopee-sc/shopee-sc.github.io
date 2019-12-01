---
title: 在 React 组件中如何向 props.children 传递数据?
author: Victor Wang
authorURL: https://github.com/beizhedenglong
---

在 `React` 组件中向 `props.children` 传递数据是设计 `ButtonGroup`/`CheckboxGroup` 等组件时常用的技巧，
我们都知道在 `React` 组件中向子组件传递数据很容易，但是如何向  `props.children` 传递数据呢？

<!--truncate-->

## 向子组件传递数据
向子组件传递数据很容易，我们只需要将数据放到子组件的 `props` 里就行了，例如：
```js

const Checkbox = (props) => {
  const { children, ...rest } = props
  return (
    <label>
      <input type="checkbox" {...rest} />
      {children}
    </label>
  )
}
const CheckboxGroup = (props) => {
  const { selected = [], group = [] } = props
  return (
    <Fragment>
      {
        group.map(
          value => (
            <Checkbox
              checked={selected.indexOf(value) > -1}
              key={value}
            >
              {value}
            </Checkbox>
          ),
        )
      }
    </Fragment>
  )
}


render(
  <div>
    <CheckboxGroup
      group={[1, 2, 3]}
      selected={[1, 3]}
    />
  </div>,
  document.getElementById('app'),
)
```
在 `CheckboxGroup` 组件中，我们把 `checked` 以及 `children` 传给了子组件`Checkbox`。这样的确是可行的，但是这样会导致父组件的配置越来越臃肿，因为我们需要在 `CheckboxGroup` 上设置一些额外的配置来控制子组件 `Checkbox` 的样式与行为。为了避免这种情况，我们需要将 `CheckboxGroup` 和 `Checkbox` 拆开，`Checkbox` 的 `props` 用来控制自己独立的样式、行为等，`CheckboxGroup` 的 `props` 主要用来控制子组件是不是全选、哪些选中、自己的样式等，主流的组件库通常这样设计这类组件的API的：
```
  <CheckboxGroup>
    <Checkbox>1</Checkbox>
    <Checkbox checked>2</Checkbox>
    <Checkbox>3</Checkbox>
  </CheckboxGroup>
```
这样使用起来更加灵活直观。为了达到这种效果，我们就需要在 `CheckboxGroup` 内向 `props.children` 传递数据。下面我们来看看有哪些常用的方法吧！

## `React.cloneElement`
我们可以借助一个  `React` 的一个顶层 `API`(`React.CloneElement`) 来动态的修改 `children` 的 `props`:
```js
const Checkbox = (props) => {
  const { children, ...rest } = props
  return (
    <label>
      <input type="checkbox" {...rest} />
      {children}
    </label>
  )
}
const CheckboxGroup = (props) => {
  const { selected = [], children } = props
  return (
    <Fragment>
      {
        // children 不是数组我们需要用 React.Children.map 来遍历
        // 或者把它转成数组
        React.Children.map(children, (child) => {
          if (!React.isValidElement(child)) {
            return null
          }
          // 这里我们通常还会判断 child 的类型来确定是不是要传递相应的数据，这里我就不做了
          const childProps = {
            ...child.props,
            checked: selected.findIndex(
              value => value.toString() === child.props.children,
            ) > -1,
          }
          return React.cloneElement(child, childProps)
        })
      }
    </Fragment>
  )
}

render(
  <div>
    <CheckboxGroup
      selected={[1, 2]}
    >
      <Checkbox>1</Checkbox>
      <Checkbox>2</Checkbox>
      <Checkbox>3</Checkbox>
    </CheckboxGroup>
  </div>,
  document.getElementById('app'),
)
```
这种做法有一个缺点，就是 `children` 不能嵌套，像下面这样就会失效：
```js
<CheckboxGroup
  selected={[1, 2]}
>
  <Fragment>
    <Checkbox>1</Checkbox>
    <Checkbox>2</Checkbox>
    <Checkbox>3</Checkbox>
  </Fragment>
</CheckboxGroup>
```
因为这时我们修改的就是 `Fragment` 的 `props` 了，而不是 `Checkbox` 的 `props`。下面我们来看看另外一种可以摆脱这种困扰的方法。

## `renderProps/renderCallback`
这里我们的 `CheckboxGroup` 可以接受一个函数，把子元素需要的数据放在该函数的参数列表里，然后动态的渲染子元素。我们直接来看实现：

```js
const Checkbox = (props) => {
  const { children, ...rest } = props
  return (
    <label>
      <input type="checkbox" {...rest} />
      {children}
    </label>
  )
}
const CheckboxGroup = (props) => {
  const { selected = [], children } = props
  return (
    <Fragment>
      {
        children(selected)
      }
    </Fragment>
  )
}

render(
  <div>
    <CheckboxGroup
      selected={[1, 2]}
    >
      {
        selected => [1, 2, 3].map(value => (
          <Checkbox
            key={value}
            checked={selected.indexOf(value) > -1}
          >
            {value}
          </Checkbox>
        ))
      }
    </CheckboxGroup>
  </div>,
  document.getElementById('app'),
)
```
这种方法可以摆脱 `cloneElement` 带来的嵌套失效的问题，但是如果使用这种方式不注意拆分的话容易写出下面这种嵌套很深的代码：
```js
<A>
  {() => (
    <B>
      {() => (
        <C>
          {() => <D />}
        </C>
      )}
    </B>
  )}
</A>
```
有没有什么更好的办法呢？有的，我们可以借助 `Context` 来共享数据。

## `Context`
当然使用 `Context` 或者其它状态管理工具逻辑都是一样的，我就不多说了。
```js
const checkboxContext = createContext([])

const Checkbox = (props) => {
  const { children, ...rest } = props
  const selected = React.useContext(checkboxContext)
  return (
    <label>
      <input
        type="checkbox"
        checked={selected.findIndex(
          value => value.toString() === children,
        ) > -1}
        {...rest}
      />
      {children}
    </label>
  )
}

const CheckboxGroup = (props) => {
  const { selected = [], children } = props
  const { Provider } = checkboxContext
  return (
    <Provider value={selected}>
      {children}
    </Provider>
  )
}

render(
  <div>
    <CheckboxGroup
      selected={[1, 3]}
    >
      <Checkbox>1</Checkbox>
      <Checkbox>2</Checkbox>
      <Checkbox>3</Checkbox>
    </CheckboxGroup>
  </div>,
  document.getElementById('app'),
)

```

这里我们就不会有嵌套带来的各种各样的问题。

本篇文章介绍了一些向 `props.children` 传递数据的常用方式，简单地分析了它们的优秀缺点，大家可以根据场景灵活选择。当然这类问题的本质还是如何优雅的解决 `React` 里各种各样的数据传递问题。

最后，希望本篇文章对大家有所帮助。