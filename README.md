
&nbsp; 

## 一 可预测的函数式

这是一个在移动端用 Redux 的状态管理模式构建出来的[[单向数据流动的函数式 View Controller]]( https://onevcat.com/2017/07/state-based-viewcontroller/). 

<img src="https://onevcat.com/assets/images/2017/view-controller-states.svg" width="600"/>

```swift
// 图中的 Store 可以对照着 redux 的 createStore 来看
class Store<A: ActionType, S: StateType, C: CommandType> {
    // CommandType 是对副作用的抽象化, 同时将 action 从副作用中解脱出来.
    let reducer: (_ state: S, _ action: A) -> (S, C?)
    var subscriber: ((_ state: S, _ previousState: S, _ command: C?) -> Void)?
    var state: S

    init(reducer:@escaping (S, A)->(S, C?), initialState: S) {
        self.reducer = reducer
        self.state = initialState
    }
    
    func subscribe(_ handler: @escaping (S, S, C?) -> Void) {
        self.subscriber = handler
    }
    func unsubscribe(){     
        self.subscriber = nil
    }
    func dispatch(_ action: A){
        let previousState = state
        let (nextState, command) = reducer(state, action)
        state = nextState
        /*
         不同于 redux 用 action creator 来隔离副作用;
         这里的副作用如异步请求是交给订阅了 command 的 subscriber 来做的, 
         command 的闭包接收请求返回的数据再 dispatch 给 reducer,
         而订阅了 nextState 的 subscriber 才是负责更新 UI 的.
        */
        subscriber?(state, previousState, command)
    }
}
```

隔离副作用来保证 `reducer` 函数的纯粹是数据可回溯、可预测的关键, redux 的 `action creator` 和 Command 都是这个用途.  

```swift
  /*
    测试中, `Command.loadToDos` 的 handler 充当了天然的 `stub`, 
    通过一组 dummy 数据 (["2", "3"]) 就能检查 store 中的状态是否符合预期，
    同时又以同步的方式测试了异步加载的过程.
  */
    let initState = TableViewController.State()
    let (_, command) = controller.reducer(initState, .loadToDos)
    XCTAssertNotNil(command)
    switch command! {
    case .loadToDos(let handler):
        handler(["2", "3"])
        XCTAssertEqual(controller.store.state.dataSource.todos, ["2", "3"])
    }
```

&nbsp; 

redux 另外一种隔离副作用的方法是 `中间件`, createStore 的第三个参数 `applyMiddleware` 可以重写 dispatch , 使得 action 在进入 dispatch 之前要先经过中间件的处理.


<img src="https://user-gold-cdn.xitu.io/2018/12/16/167b79c4d7931231?imageView2/0/w/1280/h/960/ignore-error/1" width="600"/>

```javascript
// 中间件要先校验 action , 符合条件的处理后要再 dispatch 出去一个新的 action ; 而校验未通过的 action 会传给下一个中间件.
// 所以 action、dispatch、next 是中间件必需的参数.
const reduxArray = ({ dispatch, getState }) => next => action => {
  if (Array.isArray(action)) {
    return action.forEach(act => dispatch(act))
  }
  return next(action)
}
const reduxThunk = ({ dispatch, getState }) => next => action => {
  if (typeof action === 'function') {
    return action(dispatch, getState)
  }
  return next(action)
}

// applyMiddleware 要把中间件像这样垒起来.
const reduxThunk = ({ dispatch, getState }) => next => action => {
  // reduxThunk 的校验和处理动作
  ... ...
  return action => {
    // 下游 reduxArray 的校验和处理动作
    ... ...
    return next1(action) // 这是reduxArray的, 还可以继续向下游展开直到 dispatch 的 action => {}. 
  }
}

// 中间件 reduxThunk 的返回值从外部看是 reduxThunk 的参数 next , 从内部看则是 reduxArray(next1) 的返回值.
// 这个逻辑可以用 reduce 实现.
export function compose(...fns) {
  if (fns.length === 0) return arg => arg
  if (fns.length === 1) return fns[0]
  // 数组 fns 就是 middlewares:[ next => action=> { } ], args 是 dispatch
  return fns.reduce((res, cur) => (...args) => res(cur(...args))) 
}

export function applyMiddleware(...middlewares) {
  return createStore => reducer => {
    const store = createStore(reducer)
    let { getState, dispatch } = store
    const params = {
      getState: getState,
      dispatch: (...args) => dispatch(...args)
    }
    const middlewareArr = middlewares.map(middleware => middleware(params))
    dispatch = compose(...middlewareArr)(dispatch);
    return { ...store, dispatch }
  }
}
```

&nbsp;

## 二 可分离的响应式

&nbsp;

可以说 ` if语句 + 处理副作用的 action creator = 中间件 ` , 而 `redux-observable` 中间件借助 RxJS 强大的异步和转换能力在这两个要素上都有着极其灵活的可操作性. 

```javascript
const fetchUser = username => ({ type: FETCH_USER, payload: username });
const fetchUserFulfilled = payload => ({ type: FETCH_USER_FULFILLED, payload });
/*
  首先要把 action 看做是时间维度上的集合 action$ 
  redux-observable 的 epic 函数接收一个 action$ , 再返回一个 action$, 内部则是中间件的业务逻辑.
  如果 action$ 可以继续传入 reducer 中, 就能实现 stream 版的 applyMiddleware, 即
  epic(action$, state$).scan(reducer).do(state => getState()), 它等价于:
  epic(action$, state$).subscribe(reactiveStore.dispatch) + createReactiveStore
*/ 
const fetchUserEpic = action$ => action$.pipe(
  ofType(FETCH_USER), //  if语句
  mergeMap(action =>  //  处理副作用的 action creator
    ajax.getJSON(`https://api.github.com/users/${action.payload}`).pipe(
      map(response => fetchUserFulfilled(response))
    )
  )
);
dispatch(fetchUser('torvalds'));
```

```javascript
const createReactiveStore = (reducer, initialState) => {
  const action$ = new Subject();
  let currentState = initialState;
  /*
   state 也是一个受 action 作用而不断累计的变量，scan 可以向下游传递 state 的每个累计值;
   操作符 reduce 与 scan 唯一的区别是: reduce 只会传递一个最终的累计值, 它的上游必须是有限的数据.
  */
  const store$ = action$.startWith(initialState).scan(reducer).do(state => {
    currentState = state
  });

  return {
    dispatch: (action) => {
      return action$.next(action)
    },
    getState: () => currentState,
    subscribe: (func) => {
      store$.subscribe(func);
    }
  }
}
```

&nbsp;

RxJS 项目在测试时也会使用 redux-observable 这种利用响应式分离关注点的方式将一些无关的外部逻辑隔离在 "epic" 函数之外, 来提高代码的可测试性.

<img src="http://img.wwery.com/tourist/a13320109095059.jpg" width="500"/>

```javascript
//生产者
const plus$ = () => {
  return Rx.Observable.fromEvent(document.querySelector('#plus'), 'click');
}
//观察者
const observer = {
  next: currentCount => {
    document.querySelector('#count').innerHTML = currentCount; 
  }
};
//处理业务逻辑的纯函数 
const counterPipe = (plus$, minus$) => {
  return Rx.Observable.merge(plus$.mapTo(1), minus$.mapTo(-1))
          .scan((count, delta) => count + delta, 0)
} 
/*
可测试性体现在如下方面:
·可以一次只测试一个功能 
·可以很容易制造各种测试前提条件
·可以很容易提高代码的测试覆盖率
·可以很容易模拟被测对象依赖的模块
*/
describe('Counter', () => {
  test('should add & subtract count on source', () => {
    const plus =     '^-a------|'; 
    const minus =    '^---c--d--|'; 
    const expected = '--x-y--z--|';
    const result$ = counterPipe(hot(plus), hot(minus));
    expectObservable(result$).toBe(expected, { x: 1, y: 0, z: -1, });
  }); 
});
```

&nbsp;

**Flutter** 的 `epic` 模式 ---- **Bloc (Business Logic Component)**. 

**Dart** 内置了两种对异步的支持: Future 的 `async + await` 和 Stream 的 `async* + yield`.(Stream 具备了 Observable 所需的 迭代器模式 `yield` 和 观察者模式 `listen` ).

> 所谓迭代器模式就是通过一些通用接口(getCurrent, moveToNext, isDone)来遍历一些复杂的、未知的数据集合; 观察者模式不需要这些**拉取**数据的接口遍历数据, 因为订阅了 publisher 之后, 无论数据是同步还是异步产生的, 都会自动**推送**给 observer .

&nbsp;

图中做为生产者的 sink 可以向 `Bloc` 内部监听它的 stream 传输数据; 再由另一个 stream (因为是不同 StreamController 创建的)作为观察者将处理好的数据传给它的 StreamBuilder 并同步更新这个部件.

<img src="https://upload-images.jianshu.io/upload_images/4044518-e2efb6e9dc3c1dbe.png?imageMogr2/auto-orient/strip|imageView2/2/w/561" width="500" />






