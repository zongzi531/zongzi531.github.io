---
title: TypeScript 初学之路
date: 2018-05-08 18:56:44
categories: "TypeScript"
comments: true
tags:
- TypeScript
---

<!-- no node -->

<!-- more -->

>To Do List一直是我学习框架开始的地方，最早写To Do List这个demo是在初学Vue的时候。
>时间过得很快，在我学习React的时候，我直接将这个Vue demo进行了改写。
>现在我重新将这个demo从JavaScript改写成TypeScript，在此分享一些学习总结。

此次，仍然使用create-react-app脚手架创建React项目，按照TypeScript官网推荐，你可以使用以下命令完成项目创建。

```bash
# Install create-react-app
$ npm install -g create-react-app

# Create our new project
$ create-react-app my-app --scripts-version=react-scripts-ts
```

更多详细操作可以参见[TypeScript-React-Starter](https://github.com/Microsoft/TypeScript-React-Starter)。

## 泛型参数

泛型参数类型是用来约束类的成员/方法参数类型。

```typescript
interface Component<P = {}, S = {}, SS = any> extends ComponentLifecycle<P, S, SS> { }
class Component<P, S> {
    constructor(props: P, context?: any);

    // We MUST keep setState() as a unified signature because it allows proper checking of the method return type.
    // See: https://github.com/DefinitelyTyped/DefinitelyTyped/issues/18365#issuecomment-351013257
    // Also, the ` | S` allows intellisense to not be dumbisense
    setState<K extends keyof S>(
        state: ((prevState: Readonly<S>, props: P) => (Pick<S, K> | S | null)) | (Pick<S, K> | S | null),
        callback?: () => void
    ): void;

    forceUpdate(callBack?: () => void): void;
    render(): ReactNode;

    // React.Props<T> is now deprecated, which means that the `children`
    // property is not available on `P` by default, even though you can
    // always pass children as variadic arguments to `createElement`.
    // In the future, if we can define its call signature conditionally
    // on the existence of `children` in `P`, then we should remove this.
    props: Readonly<{ children?: ReactNode }> & Readonly<P>;
    state: Readonly<S>;
    context: any;
    refs: {
        [key: string]: ReactInstance
    };
}
```

查看`Component`源码你会发现泛型参数`P`对应的就是`props`，泛型参数`S`对应的就是`state`。

所以在写`class`组件时，都会以以下格式进行定义：

```typescript
export default class App extends Component<IPropTypes, IStateTypes> {}
```

## 接口

来回顾一下，之前我们写React会使用`prop-types`来约束`props`的参数类型，但是现在我们可以放弃他，尽情发挥TypeScript的类型检查功能。

我们可以使用接口来约束`props`的类型，这同样也受用于`state`。

同样在写组件时，你会发现通用组件下会存在可选参数，那么写接口时就会使用到可选属性来解决这个问题。那么如果某些组件没有可选属性时候，你就不需要再使用`defaultProps`来设置`props`的默认值了，因为TypeScript的类型会直接抛出错误。

接口也提供导出导入功能，方便我们在开发的时候共用接口或者继承接口。

## 基础类型

期初我在定义数据类型的时候，不确定的我都会定义成`any`，当然我知道这很不优雅，也不是我的代码风格，所以我不会就此罢休。

我尝试着去把所有的不确定都换成确定。想想如果将所有的数据类型都定义成`any`，那么就失去了TypeScript的类型检查功能了，非常糟糕。

所以我在写泛型参数`props`和`state`时，如果存在没有属性的情况下，我会选择使用空对象类型`{}`，使得我的程序更可控。

## 事件类型

在写到表单内容时，会有一些需要使用到`event`的地方，这次我使用了React提供的事件类型。

```typescript
interface SyntheticEvent<T> {
    bubbles: boolean;
    /**
     * A reference to the element on which the event listener is registered.
     */
    currentTarget: EventTarget & T;
    cancelable: boolean;
    defaultPrevented: boolean;
    eventPhase: number;
    isTrusted: boolean;
    nativeEvent: Event;
    preventDefault(): void;
    isDefaultPrevented(): boolean;
    stopPropagation(): void;
    isPropagationStopped(): boolean;
    persist(): void;
    // If you thought this should be `EventTarget & T`, see https://github.com/DefinitelyTyped/DefinitelyTyped/pull/12239
    /**
     * A reference to the element from which the event was originally dispatched.
     * This might be a child element to the element on which the event listener is registered.
     *
     * @see currentTarget
     */
    target: EventTarget;
    timeStamp: number;
    type: string;
}
```

以上是`SyntheticEvent<T>`接口源码，React已经将原生事件需要部分进行了合成，在此基础上有更细化的接口：`FocusEvent<T>`、`ChangeEvent<T>`、`KeyboardEvent<T>`。

这里必须要很强势的安利一波VSCode，他让我的TypeScript开发体验变得如此之优雅，让我能够更好的学习TypeScript。

## TSLint

代码舒适区这个问题，我选择适当的妥协，我选择适当的适应规则让我成长，而不是一味的寻找舒适区。

列举一下这个demo中我配置的TSLint：

1. `"semicolon": [true, "never"]`：一如既往，我选择不写分号。
2. `"object-literal-sort-keys": false`：我关闭了短属性名提醒。
3. `"ordered-imports": false`：我关闭了`import`的约束，使我可以导入CSS文件。

但是比如定义接口时必须以大写字母`I`打头的这类约束我并没有关闭，我认为这些是需要我去适应的。

## 遇到的一些问题

1. Issues [Duplicate identifier with @types/node 10.0](https://github.com/DefinitelyTyped/DefinitelyTyped/issues/25342)。

    遇到此问题，之前我是将node版本降级至@types/node@9.6.7，但是目前@types/node@10.0.2已经修复该问题。

2. 本demo会使用到react-dnd，但是使用TypeScript时，如果继续使用修饰器的写法，将无法继续进行下去。

    我查阅Issues后选择退而求其次，使用高阶组件写法来解决此问题。
    相关Issues：[#782](https://github.com/react-dnd/react-dnd/issues/782) [#785](https://github.com/react-dnd/react-dnd/issues/785)

---

本文中提及源码版本：**@types/react@16.3.13**

GitHub仓库： [zongzi531/react-to-do-list](https://github.com/zongzi531/react-to-do-list)
