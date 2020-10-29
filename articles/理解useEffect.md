# 理解useEffect

## 前言

React 16.7的Hooks已经发布了相当长的一段时间了，这无疑是React发展史上一个转折性突破。

Hooks API 无论从简洁程度，还是使用深度角度来看，都大大优于之前生命周期的 API，所以必须反复理解，反复实践，否则只能停留在表面原地踏步。

比较难理解的是useEffect这个方法，React核心开发者Dan写了一篇文章[《 a-complete-guide-to-useeffect 》](https://overreacted.io/a-complete-guide-to-useeffect/)讲述useEffect的使用。原文很长也写的非常好，本文是笔者对原文的解读，希望对你有帮助。

## 概述

**unLearning，也就是学会忘记。你之前的学习经验会阻碍你进一步学习。** 阅读开始之前，可以先忘掉Class，忘掉生命周期，重新认识 Function Component 的思维方式。

**Function Component 是更彻底的状态驱动抽象，甚至没有 Class Component 生命周期的概念，只有一个状态，而 React 负责同步到 DOM。** 这是理解 Function Component 以及 useEffect 的关键。

要说清楚 useEffect，最好先从 Render 概念开始理解。

### 每次 Render 都有自己的 Props 与 State

可以认为每次 Render 的内容都会形成一个快照并保留下来，因此当状态变更而 Rerender 时，就形成了 N 个 Render 状态，而每个 Render 状态都拥有自己固定不变的 Props 与 State。

看下面的 `count`：

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>Click me</button>
    </div>
  );
}
```

在每次点击时，`count` 只是一个不会变的常量，存在于每次 Render 中。

初始状态下 `count` 值为 0，而随着按钮被点击，在每次 Render 过程中，`count` 的值都会被固化为 1、2、3：

```jsx
// During first render
function Counter() {
  const count = 0; // Returned by useState()
  // ...
  <p>You clicked {count} times</p>;
  // ...
}

// After a click, our function is called again
function Counter() {
  const count = 1; // Returned by useState()
  // ...
  <p>You clicked {count} times</p>;
  // ...
}

// After another click, our function is called again
function Counter() {
  const count = 2; // Returned by useState()
  // ...
  <p>You clicked {count} times</p>;
  // ...
}
```

其实不仅是对象，函数在每次渲染时也是独立的。这就是 `Capture Value` 特性，后面遇到这种情况就不会一一展开，只描述为 “此处拥有 Capture Value 特性”。

### 每次 Render 都有自己的事件处理

解释了为什么下面的代码会输出 5 而不是 3:

```jsx
const App = () => {
  const [temp, setTemp] = React.useState(5);

  const log = () => {
    setTimeout(() => {
      console.log("3 秒前 temp = 5，现在 temp =", temp);
    }, 3000);
  };

  return (
    <div
      onClick={() => {
        log();
        setTemp(3);
        // 3 秒前 temp = 5，现在 temp = 5
      }}
    >
      xyz
    </div>
  );
};
```

在 `log` 函数执行的那个 Render `过程里，temp` 的值可以看作常量 `5`，**执行 `setTemp(3)` 时会交由一个全新的 Render 渲染**，所以不会执行 `log` 函数。**而 3 秒后执行的内容是由 `temp` 为 `5` 的那个 Render 发出的**，所以结果自然为 `5`。

原因就是 `temp`、`log` 都拥有 Capture Value 特性。

### 每次 Render 都有自己的 Effects

`useEffect` 也一样具有 Capture Value 的特性。

`useEffect` 在实际 DOM 渲染完毕后执行，那 `useEffect` 拿到的值也遵循 Capture Value 的特性：

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    document.title = `You clicked ${count} times`;
  });

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>Click me</button>
    </div>
  );
}
```

上面的 `useEffect` 在每次 Render 过程中，拿到的 `count` 都是固化下来的常量。

### 如何绕过 Capture Value

利用 `useRef` 就可以绕过 Capture Value 的特性。可以认为 `ref` 在所有 Render 过程中保持着唯一引用，因此所有对 `ref` 的赋值或取值，拿到的都只有一个最终状态，而不会在每个 Render 间存在隔离。

```jsx
function Example() {
  const [count, setCount] = useState(0);
  const latestCount = useRef(count);

  useEffect(() => {
    // Set the mutable latest value
    latestCount.current = count;
    setTimeout(() => {
      // Read the mutable latest value
      console.log(`You clicked ${latestCount.current} times`);
    }, 3000);
  });
  // ...
}
```

也可以简洁的认为，`ref` 是 Mutable 的，而 `state` 是 Immutable 的。

### 不要对 Dependencies 撒谎

如果你明明使用了某个变量，却没有申明在依赖中，你等于向 React 撒了谎，后果就是，当依赖的变量改变时，`useEffect` 也不会再次执行：

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1);
    }, 1000);
    return () => clearInterval(id);
  }, []);

  return <h1>{count}</h1>;
}
```

`setInterval` 我们只想执行一次，所以我们自以为聪明的向 React 撒了谎，将依赖写成 `[]`。

“组件初始化执行一次 `setInterval`，销毁时执行一次 `clearInterval`，这样的代码符合预期。” 你心里可能这么想。

但是你错了，由于 `useEffect` 符合 `Capture Value` 的特性，拿到的 `count` 值永远是初始化的 `0`。**相当于 `setInterval` 永远在 `count` 为 `0` 的 `Scope` 中执行，你后续的 `setCount` 操作并不会产生任何作用。**

### 诚实的代价

笔者稍稍修改了一下标题，因为诚实是要付出代价的：

```jsx
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1);
  }, 1000);
  return () => clearInterval(id);
}, [count]);
```

你老实告诉 React “嘿，等 `count` 变化后再执行吧”，那么你会得到一个好消息和两个坏消息。
好消息是，代码可以正常运行了，拿到了最新的 `count`。
坏消息有：

1. 计时器不准了，因为每次 `count` 变化时都会销毁并重新计时。
2. 频繁 生成/销毁 定时器带来了一定性能负担。

### 怎么既诚实又高效呢？

上述例子使用了 `count`，然而这样的代码很别扭，**因为你在一个只想执行一次的 Effect 里依赖了外部变量**。

既然要诚实，那只好 **想办法不依赖外部变量**：

```jsx
useEffect(() => {
  const id = setInterval(() => {
    setCount(c => c + 1);
  }, 1000);
  return () => clearInterval(id);
}, []);
```

`setCount` 还有一种函数回调模式，你不需要关心当前值是什么，只要对 “旧的值” 进行修改即可。这样虽然代码永远运行在第一次 Render 中，但总是可以访问到最新的 `state`。

### 将更新与动作解耦

你可能发现了，上面投机取巧的方式并没有彻底解决所有场景的问题，比如同时依赖了两个 `state` 的情况：

```jsx
useEffect(() => {
  const id = setInterval(() => {
    setCount(c => c + step);
  }, 1000);
  return () => clearInterval(id);
}, [step]);
```

你会发现不得不依赖 · 这个变量，我们又回到了 “诚实的代价” 那一章。当然 Dan 一定会给我们解法的。

利用 `useEffect` 的兄弟 `useReducer` 函数，将更新与动作解耦就可以了：

```jsx
const [state, dispatch] = useReducer(reducer, initialState);
const { count, step } = state;

useEffect(() => {
  const id = setInterval(() => {
    dispatch({ type: "tick" }); // Instead of setCount(c => c + step);
  }, 1000);
  return () => clearInterval(id);
}, [dispatch]);
```

这就是一个局部 “Redux”，由于更新变成了 `dispatch({ type: "tick" })` 所以不管更新时需要依赖多少变量，在调用更新的动作里都不需要依赖任何变量。 具体更新操作在 `reducer` 函数里写就可以了。[在线 Demo](https://codesandbox.io/s/xzr480k0np)。

### 将 Function 挪到 Effect 里

### 如果非要把 Function 写在 Effect 外面呢？

```jsx
function Parent() {
  const [query, setQuery] = useState("react");

  // ✅ Preserves identity until query changes
  const fetchData = useCallback(() => {
    const url = "https://hn.algolia.com/api/v1/search?query=" + query;
    // ... Fetch data and return it ...
  }, [query]); // ✅ Callback deps are OK

  return <Child fetchData={fetchData} />;
}

function Child({ fetchData }) {
  let [data, setData] = useState(null);

  useEffect(() => {
    fetchData().then(setData);
  }, [fetchData]); // ✅ Effect deps are OK

  // ...
}
```

由于函数也具有 Capture Value 特性，经过 `useCallback` 包装过的函数可以当作普通变量作为 `useEffect` 的依赖。`useCallback` 做的事情，就是在其依赖变化时，返回一个新的函数引用，触发 `useEffect` 的依赖变化，并激活其重新执行。

### useCallback 带来的好处

在 Class Component 的代码里，如果希望参数变化就重新取数，你不能直接比对取数函数的 Diff：

```jsx
componentDidUpdate(prevProps) {
  // 🔴 This condition will never be true
  if (this.props.fetchData !== prevProps.fetchData) {
    this.props.fetchData();
  }
}
```

反之，要比对的是取数参数是否变化：

```jsx
componentDidUpdate(prevProps) {
  if (this.props.query !== prevProps.query) {
    this.props.fetchData();
  }
}
```

但这种代码不内聚，一旦取数参数发生变化，就会引发多处代码的维护危机。
反观 Function Component 中利用 `useCallback` 封装的取数函数，可以直接作为依赖传入 `useEffect`，**`useEffect` 只要关心取数函数是否变化，而取数参数的变化在 `useCallback` 时关心，再配合 eslint 插件的扫描，能做到 依赖不丢、逻辑内聚，从而容易维护。**

### 更更更内聚

除了函数依赖逻辑内聚之外，我们再看看取数的全过程：
一个 Class Component 的普通取数要考虑这些点：

在 `didMount` 初始化发请求。
在 `didUpdate` 判断取数参数是否变化，变化就调用取数函数重新取数。
在 `unmount` 生命周期添加 `flag`，在 `didMount` `didUpdate` 两处做兼容，当组件销毁时取消取数。

你会觉得代码跳来跳去的，不仅同时关心取数函数与取数参数，还要在不同生命周期里维护多套逻辑。那么换成 Function Component 的思维是怎样的呢？

```jsx
function Article({ id }) {
  const [article, setArticle] = useState(null);

  // 副作用，只关心依赖了取数函数
  useEffect(() => {
    // didCancel 赋值与变化的位置更内聚
    let didCancel = false;

    async function fetchData() {
      const article = await API.fetchArticle(id);
      if (!didCancel) {
        setArticle(article);
      }
    }

    fetchData();

    return () => {
      didCancel = true;
    };
  }, [fetchArticle]);

  // ...
}
```

当你真的理解了 Function Component 理念后，就可以理解 Dan 的这句话：虽然 `useEffect` 前期学习成本更高，但一旦你正确使用了它，就能比 Class Component 更好的处理边缘情况。

`useEffect` 只是底层 API，未来业务接触到的是更多封装后的上层 API，比如 `useFetch` 或者 `useTheme`，它们会更好用。

## 总结

重新捋一下这篇文章的思路：

1. 从介绍 Render 引出 Capture Value 的特性。
1. 拓展到 Function Component 一切均可 Capture，除了 Ref。
1. 从 Capture Value 角度介绍 useEffect 的 API。
1. 介绍了 Function Component 只关注渲染状态的事实。
1. 引发了如何提高 useEffect 性能的思考。
1. 介绍了不要对 Dependencies 撒谎的基本原则。
1. 从不得不撒谎的特例中介绍了如何用 Function Component 思维解决这些问题。
1. 当你学会用 Function Component 理念思考时，你逐渐发现它的一些优势。
1. 最后点出了逻辑内聚，高阶封装这两大特点，让你同时领悟到 Hooks 的强大与优雅。

其中两个最重要的点：

1. Capture Value 特性。
2. 一致性。**将注意放在依赖上（`useEffect` 的第二个参数 `[]`），而不是关注何时触发。**
