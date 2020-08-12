### React

#### 组件

组件可以添加属性，通过this.prop.name来获取
注意： class 属性需要写成 className ，for 属性需要写成 htmlFor ，这是因为 class 和 for 是 JavaScript 的保留字。
react.createRender方法用于创建组件类，所有组件类都必须有render方法，用于输出组件。

this.props 对象的属性与组件的属性一一对应，但是有一个例外，就是 this.props.children 属性。它表示组件的所有子节点。

使用PropType类型检查：用噶就是在class中添加一个叫propTypes的属性（是一个对象）
    title: React.PropTypes.string.isRequired就表示title是一个字符串并且必须要有

使用refs.xxxx获取真实dom
需要注意的是，由于 this.refs.[refName] 属性获取的是真实 DOM ，所以必须等到虚拟 DOM 插入文档以后，才能使用这个属性
#state
当用户点击组件，导致状态变化，this.setState 方法就修改状态值，每次修改以后，自动调用 this.render 方法，再次渲染组件。
使用：添加一个class构造函数，然后在该函数中为this.state赋初值
Class 组件应该始终使用 props 参数来调用父类的构造函数。
constructor(props) {
    super(props);
    this.state = {date: new Date()};
  }

<!-- more -->

##### 正确使用state

关于setState()需要注意三点
1.不要直接修改state
  应该使用setState(),构造函数是唯一可以给this.state赋值（这里意思应该是直接赋值）的地方
2.因为 this.props 和 this.state 可能会异步更新，所以你不要依赖他们的值来更新下一个状态。
解决办法：可以让 setState() 接收一个函数而不是一个对象。这个函数用上一个 state 作为第一个参数，将此次更新被应用时的 props 做为第二个参数：
// Correct
this.setState((state, props) => ({
  counter: state.counter + props.increment
}));
3.当你调用 setState() 的时候，React 会把你提供的对象合并到当前的 state。
这是一种浅合并，只会合并或替换新传进去的属性，其他已经存在的state属性不受影响，所以我们可以分别调用setState()来单独更新属性。

#### 事件处理
语法上和DOM事件的一点不同
1.React 事件的命名采用小驼峰式（camelCase），而不是纯小写。
2.使用 JSX 语法时你需要传入一个函数作为事件处理函数，而不是一个字符串
<button onClick={activateLasers}>
3.在 React 中另一个不同点是你不能通过返回 false 的方式阻止默认行为。你必须显式的使用 preventDefault
4.使用ES6 class定义组件的时候，通常做法是将事件处理函数声明为class中的方法。但是要注意回调函数中的this问题，一般就是用class fields和bind(在构造函数里bind)
class fields貌似就是ES6中class实例属性写在顶部的用法
  class LoggingButton extends React.Component {
    // 此语法确保 `handleClick` 内的 `this` 已被绑定。
    // 注意: 这是 *实验性* 语法。
    handleClick = () => {
      console.log('this is:', this);
    }

```javascript
render() {
  return (
    <button onClick={this.handleClick}>
      Click me
    </button>
  );
}
```
  }

#### 怎么向事件处理函数传递参数呢？
  两种方法，通过箭头函数或bind
<button onClick={(e) => this.deleteRow(id, e)}>Delete Row</button>
使用箭头函数必须显示的传递事件对象e
<button onClick={this.deleteRow.bind(this, id)}>Delete Row</button>
为什么是这种形式？大概就是因为jsx语法事件处理需要传一个函数而不是一个字符串。

*问题：为什么会有这样的不同？需要思考一下。*

#### 条件渲染

很简单粗暴，就是用if。可以把需要条件显示的组件封装到一个组件中，把判断条件作为prop传入，按条件return就可以了
还有一些简单的内联显示方法：

1. &&运算符
之所以能这样做，是因为在 JavaScript 中，true && expression 总是会返回 expression, 而 false && expression 总是会返回 false。
因此，如果条件是 true，&& 右侧的元素就会被渲染，如果是 false，React 会忽略并跳过它。
2. 三目运算符

#### 阻止组件渲染
组件返回null即可。不会影响组件的生命周期

#### 列表&key

react中把数组转换为列表用的就是map方法。
const numbers = [1, 2, 3, 4, 5];
const listItems = numbers.map((number) =><li>{number}</li>
最后render渲染<ul>{listItems}</ul>

key 帮助 React 识别哪些元素改变了，比如被添加或删除。因此你应当给数组中的每一个元素赋予一个确定的标识。
const listItems = numbers.map((number) =>
    //key 应 该在数组的上下文中被指定
    <ListItem key={number.toString()}
              value={number} />
  );
通常使用来自数据id作为key
一个好的经验法则是：在 map() 方法中的元素需要设置 key 属性。

key 只是在兄弟节点之间必须唯一
数组元素中使用的 key 在其兄弟节点之间应该是独一无二的。然而，它们不需要是全局唯一的。当我们生成两个不同的数组时，我们可以使用相同的 key 值

#### 表单

受控组件:
使 React 的 state 成为“唯一数据源”。渲染表单的 React 组件还控制着用户输入过程中表单发生的操作。被 React 以这种方式控制取值的表单输入元素就叫做“受控组件”。
一般就是value属性
#状态提升
将共享属性提升至父组件的state，连同处理函数一同作为prop传入
#组合vs继承
简单来说类似vue中的插槽，别的组件在用这个组件的时候可以自定义预留空位的内容。
只有一个空位就用props.children。调用的时候该位置所有的jsx内容都会作为一个children prop传递给组件。
多个空位就自行约定{props.xxx}  然后调用的时候 < exapmple xxx={  } /> 这样传进去

为什么不用继承？
在 Facebook，我们在成百上千个组件中使用 React。我们并没有发现需要使用继承来构建组件层次的情况。
Props 和组合为你提供了清晰而安全地定制组件外观和行为的灵活方式。注意：组件可以接受任意 props，包括基本数据类型，React 元素以及函数。
如果你想要在组件间复用非 UI 的功能，我们建议将其提取为一个单独的 JavaScript 模块，如函数、对象或者类。组件可以直接引入（import）而无需通过 extend 继承它们。
### Redux
1. Store
store就是保存数据的地方，整个应用只能有一个store
createStore(reducer)用来生成store
createStore方法还可以接受第二个参数，表示 State 的最初状态。这通常是服务器给出的。
let store = createStore(todoApp, window.STATE_FROM_SERVER)

2. State
Store对象包含所有数据。如果想得到某个时点的数据，就要对 Store 生成快照。这种时点的数据集合，就叫做 State。
当前时刻的 State，可以通过store.getState()拿到。
Redux 规定， 一个 State 对应一个 View。只要 State 相同，View 就相同。你知道 State，就知道 View 是什么样，反之亦然。
3. Action
State 的变化，会导致 View 的变化。但是，用户接触不到 State，只能接触到 View。所以，State 的变化必须是 View 导致的。Action 就是 View 发出的通知，表示 State 应该要发生变化了。
Action 是一个对象。其中的type属性是必须的，表示 Action 的名称。其他属性可以自由设置。
可以这样理解，Action 描述当前发生的事情。
    ###*改变 State 的唯一办法，就是使用 Action*。它会运送数据到 Store。
可以写一个函数来生成Action
function addTodo(text) {
    return {
    type: ADD_TODO,
    text
    }
}
4. store.dispatch()
是View层发出Action的唯一方法
store.dispatch接受一个 Action 对象作为参数，将它发送出去
#5. Reducer
Store 收到 Action 以后，必须给出一个新的 State，这样 View 才会发生变化。这种 State 的计算过程就叫做 Reducer。
Reducer 是一个函数，它接受 Action 和当前 State 作为参数，返回一个新的 State。
整个应用的初始状态，可以作为 State 的默认值（传参的时候设置）
实际上reducer不用手动调用，在生成 Store 的时候，将 Reducer 传入createStore方法。这样store.dispatch方法会触发 Reducer 的自动执行。
为什么这个函数叫做 Reducer 呢？因为它可以作为数组的reduce方法的参数。
6. 纯函数
reducer需要遵守以下约束。
    不得改写参数
    不能调用系统 I/O 的API
    不能调用Date.now()或者Math.random()等不纯的方法，因为每次会得到不一样的结果
因为不能更改state，所以reducer返回的新state是一个新对象
参考写法：
// State 是一个对象
function reducer(state, action) {
    return Object.assign({}, state, { thingToChange });
    // 或者
    return { ...state, ...newState };
}

// State 是一个数组
function reducer(state, action) {
  return [...state, newItem];
}
7. store.subscribe
    设置监听函数，一旦state发生变化就自动执行这个函数

  ```javascript
  import { createStore } from 'redux';
    const store = createStore(reducer);
    store.subscribe(listener);
  ```

     很显然，在listener中写入view的更新函数（组建的render方法或者setState方法），会会实现View的自动渲染。
      store.subscribe方法返回一个函数，调用这个函数就可以解除监听。

8. Reducer的拆分/合成
    系统复杂之后，reducer必然会很庞大，我们需要拆分成一个一个小的专门处理对应功能的子reducer，然后将他们合并就可以。
    Redux 提供了一个combineReducers方法，用于 Reducer 的拆分。你只要定义各个子 Reducer 函数，然后用这个方法，将它们合成一个大的 Reducer（模块化的感觉就出来了，分出来的也可以继续分）
    combineReducer有两种写法：
    import { combineReducers } from 'redux';
    const chatReducer = combineReducers({
   chatLog,//等同于chatLog: chatLog(state.chatLog, action)
   statusMessage,
   userName
    })
    //reducer返回的是一个新的state对象
    export default todoApp;
    这种写法有一个前提，就是 *State 的属性名必须与子 Reducer 同名*。如果不同名，就要采用下面的写法。
    否则要这么写，
   a: doSomethingWithA,等同于a: doSomethingWithA(state.a, action)
    a就是state属性

##### combineReducer的实现，内部原理（以后再看）

```javascript
const combineReducers = reducers => {
  return (state = {}, action) => {
    return Object.keys(reducers).reduce(
      (nextState, key) => {
        nextState[key] = reducers[key](state[key], action)
        return nextState;
      },{})
  }
}
```

### Redux进阶：中间件和异步

#### React Hook

组件写成纯函数，需要外部功能和副作用，就用钩子把外部代码'钩'进来.
约定钩子一律用usexxxx命名
常见的默认提供的钩子

1.useState()状态钩子
const [   , setxxx  ] = useState(" ")
useState接收状态初始值，返回一个数组，数组第一个成员是一个变量（只想状态当前值）。第二个成员是一个函数（用来更新状态，约定名字叫setxxx，用法就是setxxx("新的状态值")）

2.useContext() 共享状态钩子

### 在端点实习遇到的问题

1.fields静态不更新的问题（存疑，日子太过久远，已经记不起具体的问题定位）
情景：fields= {this.fields}中依赖state中的数据，但是更新state之后组件没有重新渲染 。
原因：this.fields是静态的，state更新之后重新触发render，但是在传入props里的fields={this.fields}中的数据没有更新。
解决办法：将class中的fields改为一个方法返回数据，然后用组件的时候fields= {this.getFields(this.state.x )} 这样render会重新调用方法，获取最新数据.。

2.this.props.\_GLOBAL_
情景: 某某组件中const { userId } = this.props.\_GLBOAL_.userInfo.userId报错
原因：这个组件是外层组件里的一个子组件。redux注入的数据，子组件是接收不到的，父组件可以直接从this.props里取，而子组件需要的话，要么从props里传进去，要么就用导入@terminus/flagon/withModels。

https://www.cnblogs.com/kuaizifeng/p/9417127.html

引出的点：装饰器

@withModel其实就是装饰器

https://segmentfault.com/p/1210000009968000/read 有空好好看下

2020/4/11

做毕设的时候接触了react router的withRouter，其实也是一样的，不被Route直接跳转的组件，就一样加一下withRoute装饰一下，然后才能使用this.props.history.push

3.ref

情景：项目中需要使用echarts绘制图表，为了管理canvas，使用一个二维数组来记录ref值。

```javascript
  renderHTML() {
    this.canvas = []
    const houseResources = this.houseResourcePeriods(this.state.houseResources)
    return houseResources.map(periodization => {
      this.canvas[periodization.periodId] = []
      return (
        <div>
          <h3>{periodization.periodName}</h3>
          <div className={styles.canvasContainer}>
            {periodization.houseBlocks.map(houseBlock => (
              <div ref={ref => { this.canvas[periodization.periodId][houseBlock.blockId] = ref}} className={styles.canvas} />
            ))}
          </div>
        </div>
      )
    })
  }
// render函数里
{this.state.houseResources && this.renderHTML()}
```

折腾了很久，一开始是setState里属性名写错了(坑x1)，到最后发现就算是不调用echart也总是报一个cannot read property of undefined(后记：应当只有重置数据会出现)。

最后查了一下ref，终于知道错在哪了。

先说一下ref：

ref属性可以设置为一个回调函数，这也是官方强烈推荐的用法；这个函数执行的时机为：

- 组件被挂载后`，回调函数被立即执行，回调函数的参数为该组件的具体实例。
- `组件被卸载或者原有的ref属性本身发生变化时`，回调也会被立即执行，此时回调函数参数为`null`，以确保内存泄露

先说坑1，因为setState写错了，所以houseResures没变，this.canvas\[][]里的值每次都会被赋予一模一样的值，所以没有报cannot read property of undefined，以及不管怎么做，都是第一次渲染好的图表。后来setState改正之后，数据源变了，ref回调会被再次调用，此时this.canvas已经被我清空，houseResources也被赋予了空[]值，map函数不执行this.canvas\[periodization.periodId] 自然就成了undefined

this.canvas\[periodization.periodId][houseBlock.blockId] 就会报错。

解决办法：将二维数组改为一维就可以。

```javascript
<div ref={ref => { 
   this.canvas[`${periodization.periodId}-${houseBlock.blockId}`] = ref}}  />
```

4.this的问题

场景：一个筛选组件Filter是这么写的：

```php+HTML
<Filter config={FilterJson.form} onSubmit={this.fetchUserInfos} />
```
问题是下面的Table组件点击查询按钮之后数据没有渲染，后台是正常返回数据的。

解决：当时几乎都没注意。。满脑子想着是不是数据没有更新，根本没发现控制台报错：

setState of undefined。因为箭头函数用的太习惯了，刚巧

fetchUserInfos没有使用箭头函数写法，又是作为callback，所以this指向有问题，函数里this.setState没有执行。。。之前filter都是用的封装好的组件，传一个json配置进去就好了，从来没遇到过这种问题。以后还是得注意下