# React 函数式组件性能优化

## React优化的方向

1. **减少重新render的次数。因为进入diff算法后，判断到组件是Updating状态，如果`shouldComponentUpdate()`返回`false`,就直接停止diff，所以也不会执行render，减少性能开销**
2. **减少计算的量。主要是减少重复计算，对于函数式组件来说，每次 render 都会重新从头开始执行函数调用。**

## React.memo

首先，**如果父组件重新渲染，那么子组件也会重新渲染，不管props有没有改变，这是diff算法的特性**，所以才有了`shouldComponentUpdate`，让用户手动去调优

说到`shouldComponentUpdate`，就顺带提一下React 15.3添加的`PureComponent`，它的原理是：当组件更新时，如果组件的 `props` 和 `state` 都没发生改变，`render` 方法就不会触发，达到提升性能的目的。具体就是 `React` 自动帮我们做了一层浅比较：

```js
if (this._compositeType === CompositeTypes.PureClass) {
  shouldUpdate = !shallowEqual(prevProps, nextProps)
  || !shallowEqual(inst.state, nextState);
}
```

而 `shallowEqual` 又做了什么呢？会比较 `Object.keys(state | props)` 的长度是否一致，每一个 `key`是否两者都有，并且是否是一个引用，也就是只比较了第一层的值，确实很浅，所以深层的嵌套数据是对比不出来的。所以**state和props不能是一个引用类型，否则永远返回`true`**。

`React.memo`方法和`PureComponent`原理一样，只是前者用于函数式组件，后者用于类组件，并且第二个参数可以传入一个比较函数，用于引用类型props的比较，用法如下：

```js
function MyComponent(props) {
  /* 使用 props 渲染 */
}
function areEqual(prevProps, nextProps) {
  /*
  如果把 nextProps 传入 render 方法的返回结果与
  将 prevProps 传入 render 方法的返回结果一致则返回 true，
  否则返回 false
  */
}
exportdefault React.memo(MyComponent, areEqual);
```

## useCallback

那么如果props是一个函数呢，父组件渲染，子组件会不会渲染？答案是会。因为在函数式组件里每次重新渲染，函数组件都会重头开始重新执行，那么这两次创建的 callback 函数肯定发生了改变，所以导致了子组件重新渲染。你可能想到用上面的`React.memo`来优化，但你如何判断两个函数是否相等呢，是不是进入到知识盲区了？其实`React.memo`里对比的引用类型只能是`Array`或`Object`，是对比不了`function`的，这时就需要`useCallback`了。

```js
const callback = () => {
  doSomething(a, b);
}

const memoizedCallback = useCallback(callback, [a, b])
```

把函数以及依赖项作为参数传入 `useCallback`，它将返回该回调函数的 memoized 版本，这个 memoizedCallback 只有在依赖项有变化的时候才会更新。这样问题又回到了对比基本类型或引用类型（除function外）上了。

## useMemo

假如有一个组件

```js
function App() {
  const [num, setNum] = useState(0);

  // 一个非常耗时的一个计算函数
  // result 最后返回的值是 49995000
  function expensiveFn() {
    let result = 0;

    for (let i = 0; i < 10000; i++) {
      result += i;
    }

    console.log(result) // 49995000
    return result;
  }

  const base = expensiveFn();

  return (
    <div className="App">
      <h1>count：{num}</h1>
      <button onClick={() => setNum(num + base)}>+1</button>
    </div>
  );
}
```

这个例子功能很简单，就是点击 **+1** 按钮，然后会将现在的值(num) 与 计算函数 (expensiveFn) 调用后的值相加，然后将和设置给 num 并显示出来，在控制台会输出 `49995000`。

产生的性能问题：每次点击按钮render函数执行这个昂贵的计算逻辑，是不是可能会阻塞到页面渲染了？这时`useMemo`出场了：

```js
function computeExpensiveValue() {
  // 计算量很大的代码
  return xxx
}

const memoizedValue = useMemo(computeExpensiveValue, [a, b]);
```

useMemo 的第一个参数就是一个函数，这个函数返回的值会被缓存起来，同时这个值会作为 useMemo 的返回值，第二个参数是一个数组依赖，如果数组里面的值有变化，那么就会重新去执行第一个参数里面的函数，并将函数返回的值缓存起来并作为 useMemo 的返回值。对于上面的例子，用法就是

```js
const base = useMemo(expensiveFn, []);
```

这样无论我们点击 **+1**多少次，只会输出一次 **49995000**，这就代表 expensiveFn 只执行了一次，达到了我们想要的效果。

## 总结

对于性能瓶颈可能对于小项目遇到的比较少，毕竟计算量小、业务逻辑也不复杂，但是对于大项目，很可能是会遇到性能瓶颈的，但是对于性能优化有很多方面：网络、关键路径渲染、打包、图片、缓存等等方面，具体应该去优化哪方面还得自己去排查，本文只介绍了性能优化中的冰山一角：运行过程中 React 的优化。

1. React 的优化方向：减少 render 的次数；减少重复计算。
2. 如何去找到 React 中导致性能问题的方法，见 useCallback 部分。
3. 合理的拆分组件其实也是可以做性能优化的，你这么想，如果你整个页面只有一个大的组件，那么当 props 或者 state 变更之后，需要 reconction 的是整个组件，其实你只是变了一个文字，如果你进行了合理的组件拆分，你就可以控制更小粒度的更新。

> 合理拆分组件还有很多其他好处，比如好维护，而且这是学习组件化思想的第一步，合理的拆分组件又是一门艺术了，如果拆分得不合理，就有可能导致状态混乱，多敲代码多思考。