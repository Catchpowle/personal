---
title: Beware React.cloneElement
date: "2021-03-08T22:12:03.284Z"
description: "Why you might want to avoid this when creating your own libraries."
---

Recently I came across the following issue using a 3rd party library at work.

Below is an example component API, I've made the components and most of the prop names generic so we can focus on the issue.

```jsx
<Parent space="sm">
  <Child propOne={valueOne} propTwo={valueOne} propThree={valueThree}>
    Content
  </Child>
  <Child propOne={valueOne} propTwo={valueOne} propThree={valueThree}>
    Content
  </Child>
  <Child propOne={valueOne} propTwo={valueOne} propThree={valueThree}>
    Content
  </Child>
</Parent>
```

The library exposes a parent component that can set the space between it's children. These children also have a bunch of props that can be configured.

This 3rd party library has to cover many use cases and is therefore very flexibile in how it can configured. Within our company's component library we don't want to expose this flexibility as we are developing within the constraints of our design system. We want to wrap the child and only the allow the dynamic configuration of certain properties.

```jsx
function WrappedChild({ propOne, propTwo, children }) {
  propThree = "valueThree"

  return (
    <Child propOne={propOne} propTwo={propTwo} propThree={propThree}>
      {children}
    </Child>
  )
}
```

We know within our component library we always want propThree of the third party component to be set to a certain value and so our WrappedChild component does this for us ensuring we don't have to worry about a developer causing issues by misconfguring this.

```jsx
<Parent space="sm">
  <WrappedChild propOne={valueOne} propTwo={valueOne}>
    Content
  </WrappedChild>
  <WrappedChild propOne={valueOne} propTwo={valueOne}>
    Content
  </WrappedChild>
  <WrappedChild propOne={valueOne} propTwo={valueOne}>
    Content
  </WrappedChild>
</Parent>
```

Unfortunately when we attempt to use our components as shown above we find to our horror that the spacing isn't being applied, why is this?

Well if we look into the source code for the third party library we can see that they are doing the following:

```jsx
function Parent({ space, children }) {
  return React.Children.map(children, child => {
    React.cloneElement(child, { space })
  })
}
```

They are cloning the child component and passing the parents space property to the child so that it can handle it's own margin. In our case that child will be our WrappedChild component and as we are not passing the space prop through we end up with the spacing issue.

One fix would be to just pass the space prop through or simply spread all remaining props, however this means we're exposing props that we don't actually want the users of our component to use directly.

Another approach would be to use React's Context API as follows:

```jsx
const ChildContext = createContext({ space: "none" })

function Child({ propOne, propTwo, propThree, children }) {
  const { space } = useContext(ChildContext)

  return // Some component that makes use of the above props
}

function Parent({ space }) {
  return (
    <ChildContext.Provider value={{ space }}>{children}</ChildContext.Provider>
  )
}
```

This approach allows us to retrieve the value for space within the child component without needing to expose unnecessary props and without the worry of a wrapper component breaking functionality.
