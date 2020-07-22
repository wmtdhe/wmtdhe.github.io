版本：`React 16.13.1`

`React`批量更新原则：

同一事件中如果有多个更新，尽管他们的`event time`可能不同，但是都会被同一时间更新

`fiber`之前的`React`更新是从上到下遍历更新的，如果一个组件过于复杂，组成部分很多，则更新一次这个组件可能会花较长的时间，如更新这个组件花费了1s，在这1s内，主线程完全被`React`更新霸占，导致别的交互、渲染什么的出现了卡顿等不流畅现象。

`fiber`的出现对时间进行了**分片** —— 1s有60帧，1帧为16.66ms，如果在1帧中执行了别的**渲染**和**用户输入响应**之后还有剩余时间，`React`则会利用剩余时间对组件进行更新。因为`fiber`，`React`的更新可以暂停，恢复以及根据`priority 优先级`交出当前更新执行权到更高的优先级的更新上。从而使得`React`不长时间霸占主线程，优化性能。

`requestHostCallback`安排一个`host的回调`，通过`postMessage`

`fiber`维护了两个`state tree`，一个是**上次更新的**，一个是**正在进行变更的**，从而使得两个可以互换，舍弃，继续更新。

#### packages/react-reconciler/src/ReactFiberClassComponent.new.js
```javascript
const classComponentUpdater = {
  isMounted,
  enqueueSetState(inst, payload, callback) { // 把 setState 排入队列
    const fiber = getInstance(inst);
    const eventTime = requestEventTime();
    const suspenseConfig = requestCurrentSuspenseConfig();
    const lane = requestUpdateLane(fiber, suspenseConfig);

    // update object有eventTime，lane，suspenseConfig，payload，callback
    const update = createUpdate(eventTime, lane, suspenseConfig); // 返回的是一个update object
    update.payload = payload; // 添加更新的 data payload
    if (callback !== undefined && callback !== null) { // 添加更新后的回调函数
      if (__DEV__) {
        warnOnInvalidCallback(callback, 'setState');
      }
      update.callback = callback;
    }

    enqueueUpdate(fiber, update);  // 将update添加到fiber的updateQueue中
    scheduleUpdateOnFiber(fiber, lane, eventTime); // 调度fiber上的update
  },
  enqueueReplaceState(inst, payload, callback) {
    const fiber = getInstance(inst);
    const eventTime = requestEventTime();
    const suspenseConfig = requestCurrentSuspenseConfig();
    const lane = requestUpdateLane(fiber, suspenseConfig);

    const update = createUpdate(eventTime, lane, suspenseConfig);
    update.tag = ReplaceState;
    update.payload = payload;

    if (callback !== undefined && callback !== null) {
      if (__DEV__) {
        warnOnInvalidCallback(callback, 'replaceState');
      }
      update.callback = callback;
    }

    enqueueUpdate(fiber, update);
    scheduleUpdateOnFiber(fiber, lane, eventTime);
  },
  enqueueForceUpdate(inst, callback) {
    const fiber = getInstance(inst);
    const eventTime = requestEventTime();
    const suspenseConfig = requestCurrentSuspenseConfig();
    const lane = requestUpdateLane(fiber, suspenseConfig);

    const update = createUpdate(eventTime, lane, suspenseConfig);
    update.tag = ForceUpdate;

    if (callback !== undefined && callback !== null) {
      if (__DEV__) {
        warnOnInvalidCallback(callback, 'forceUpdate');
      }
      update.callback = callback;
    }

    enqueueUpdate(fiber, update);
    scheduleUpdateOnFiber(fiber, lane, eventTime);
  },
};


```

#### ReactQueueUpdate.new.js
```javascript
export function enqueueUpdate<State>(fiber: Fiber, update: Update<State>) {
  const updateQueue = fiber.updateQueue;
  if (updateQueue === null) {
    // Only occurs if the fiber has been unmounted.
    return;
  }

  const sharedQueue: SharedQueue<State> = (updateQueue: any).shared;
  const pending = sharedQueue.pending;
  if (pending === null) {
    // This is the first update. Create a circular list.
    update.next = update;
  } else {
    update.next = pending.next;
    pending.next = update;
  }
  sharedQueue.pending = update;

  if (__DEV__) {
    if (
      currentlyProcessingQueue === sharedQueue &&
      !didWarnUpdateInsideUpdate
    ) {
      console.error(
        'An update (setState, replaceState, or forceUpdate) was scheduled ' +
          'from inside an update function. Update functions should be pure, ' +
          'with zero side-effects. Consider using componentDidUpdate or a ' +
          'callback.',
      );
      didWarnUpdateInsideUpdate = true;
    }
  }
}

```

#### packages/react-reconciler/src/ReactFiberWorkLoop.new.js
```javascript
export function scheduleUpdateOnFiber(
  fiber: Fiber,
  lane: Lane,
  eventTime: number,
) {
  checkForNestedUpdates();
  warnAboutRenderPhaseUpdatesInDEV(fiber);

  const root = markUpdateLaneFromFiberToRoot(fiber, lane);
  if (root === null) {
    warnAboutUpdateOnUnmountedFiberInDEV(fiber);
    return null;
  }
  // TODO: requestUpdateLanePriority also reads the priority. Pass the
  // priority as an argument to that function and this one.
  const priorityLevel = getCurrentPriorityLevel();

  if (lane === SyncLane) { // 同步更新
    if (
      // Check if we're inside unbatchedUpdates 如果是非批量更新上下文
      (executionContext & LegacyUnbatchedContext) !== NoContext &&
      // Check if we're not already rendering 如果不是在渲染或提交上下文
      (executionContext & (RenderContext | CommitContext)) === NoContext
    ) {
      // Register pending interactions on the root to avoid losing traced interaction data.
      schedulePendingInteractions(root, lane);

      // This is a legacy edge case. The initial mount of a ReactDOM.render-ed
      // root inside of batchedUpdates should be synchronous, but layout updates
      // should be deferred until the end of the batch.
      performSyncWorkOnRoot(root);
    } else {
      ensureRootIsScheduled(root, eventTime);
      schedulePendingInteractions(root, lane);
      if (executionContext === NoContext) { 
        // Flush the synchronous work now, unless we're already working or inside
        // a batch. This is intentionally inside scheduleUpdateOnFiber instead of
        // scheduleCallbackForFiber to preserve the ability to schedule a callback
        // without immediately flushing it. We only do this for user-initiated
        // updates, to preserve historical behavior of legacy mode.
        flushSyncCallbackQueue();
      }
    }
  } else {
    // Schedule a discrete update but only if it's not Sync.
    if ( // 独立事件上下文并且优先级为ImmediateScheduler或者UserBlockingScheduler
      (executionContext & DiscreteEventContext) !== NoContext && 
      // Only updates at user-blocking priority or greater are considered
      // discrete, even inside a discrete event.
      (priorityLevel === UserBlockingSchedulerPriority ||
        priorityLevel === ImmediateSchedulerPriority)
    ) {
      // This is the result of a discrete event. Track the lowest priority
      // discrete update per root so we can flush them early, if needed.
      if (rootsWithPendingDiscreteUpdates === null) {
        rootsWithPendingDiscreteUpdates = new Set([root]);
      } else {
        rootsWithPendingDiscreteUpdates.add(root);
      }
    }
    // Schedule other updates after in case the callback is sync. 其他上下文的更新，如批量更新上下文 batchedContext
    ensureRootIsScheduled(root, eventTime);
    schedulePendingInteractions(root, lane);
  }
  // We use this when assigning a lane for a transition inside
  // `requestUpdateLane`. We assume it's the same as the root being updated,
  // since in the common case of a single root app it probably is. If it's not
  // the same root, then it's not a huge deal, we just might batch more stuff
  // together more than necessary.
  mostRecentlyUpdatedRoot = root;
}
```

#### fiber
> A Fiber is work on a Component that needs to be done or was done. There can
> be more than one per component.
>
> fiber是组件上需要做的或者已经完成的工作，一个组件可能有多个fiber
> 
> 任务有优先级，高优先级会打断正在进行的低优先级任务

**Fiber数据结构**
```javascript
export type Fiber = {|
  // These first fields are conceptually members of an Instance. This used to
  // be split into a separate type and intersected with the other Fiber fields,
  // but until Flow fixes its intersection bugs, we've merged them into a
  // single type.

  // An Instance is shared between all versions of a component. We can easily
  // break this out into a separate object to avoid copying so much to the
  // alternate versions of the tree. We put this on a single object for now to
  // minimize the number of objects created during the initial render.

  // Tag identifying the type of fiber.
  tag: WorkTag,

  // Unique identifier of this child.
  key: null | string,

  // The value of element.type which is used to preserve the identity during
  // reconciliation of this child.
  elementType: any,

  // The resolved function/class/ associated with this fiber.
  type: any,

  // The local state associated with this fiber. 通常指的DOM元素
  /** ！！！批量更新之前恢复的状态来源！！！ **/
  stateNode: any,

  // Conceptual aliases
  // parent : Instance -> return The parent happens to be the same as the
  // return fiber since we've merged the fiber and instance.

  // Remaining fields belong to Fiber

  // The Fiber to return to after finishing processing this one.
  // This is effectively the parent, but there can be multiple parents (two)
  // so this is only the parent of the thing we're currently processing.
  // It is conceptually the same as the return address of a stack frame.
  return: Fiber | null,

  // Singly Linked List Tree Structure. 单链表
  child: Fiber | null,
  sibling: Fiber | null,
  index: number,

  // The ref last used to attach this node.
  // I'll avoid adding an owner field for prod and model that as functions.
  ref:
    | null
    | (((handle: mixed) => void) & {_stringRef: ?string, ...})
    | RefObject,

  // Input is the data coming into process this fiber. Arguments. Props.
  pendingProps: any, // This type will be more specific once we overload the tag. 还未更新的props
  memoizedProps: any, // The props used to create the output. 之前的props

  // A queue of state updates and callbacks. 状态更新和回调的队列
  updateQueue: mixed,

  // The state used to create the output 之前的状态
  memoizedState: any,

  // Dependencies (contexts, events) for this fiber, if it has any
  dependencies: Dependencies | null,

  // Bitfield that describes properties about the fiber and its subtree. E.g.
  // the ConcurrentMode flag indicates whether the subtree should be async-by-
  // default. When a fiber is created, it inherits the mode of its
  // parent. Additional flags can be set at creation time, but after that the
  // value should remain unchanged throughout the fiber's lifetime, particularly
  // before its child fibers are created.
  mode: TypeOfMode,

  // Effect
  effectTag: SideEffectTag,

  // Singly linked list fast path to the next fiber with side-effects.
  // 单链，连接下一个有副作用的fiber（需要进行DOM操作的节点）
  nextEffect: Fiber | null,

  // The first and last fiber with side-effect within this subtree. This allows
  // us to reuse a slice of the linked list when we reuse the work done within
  // this fiber.
  firstEffect: Fiber | null,
  lastEffect: Fiber | null,

  lanes: Lanes,
  childLanes: Lanes,

  // This is a pooled version of a Fiber. Every fiber that gets updated will
  // eventually have a pair. There are cases when we can clean up pairs to save
  // memory if we need to.
  // 每个会更新的fiber都有一个相应的pair
  alternate: Fiber | null,

  // Time spent rendering this Fiber and its descendants for the current update.
  // This tells us how well the tree makes use of sCU for memoization.
  // It is reset to 0 each time we render and only updated when we don't bailout.
  // This field is only set when the enableProfilerTimer flag is enabled.
  actualDuration?: number,

  // If the Fiber is currently active in the "render" phase,
  // This marks the time at which the work began.
  // This field is only set when the enableProfilerTimer flag is enabled.
  actualStartTime?: number,

  // Duration of the most recent render time for this Fiber.
  // This value is not updated when we bailout for memoization purposes.
  // This field is only set when the enableProfilerTimer flag is enabled.
  selfBaseDuration?: number,

  // Sum of base times for all descendants of this Fiber.
  // This value bubbles up during the "complete" phase.
  // This field is only set when the enableProfilerTimer flag is enabled.
  treeBaseDuration?: number,

  // Conceptual aliases
  // workInProgress : Fiber ->  alternate The alternate used for reuse happens
  // to be the same as work in progress.
  // __DEV__ only
  _debugID?: number,
  _debugSource?: Source | null,
  _debugOwner?: Fiber | null,
  _debugIsCurrentlyTiming?: boolean,
  _debugNeedsRemount?: boolean,

  // Used to verify that the order of hooks does not change between renders.
  _debugHookTypes?: Array<HookType> | null,
|};
```
React对于事件的处理有6种优先级

1. ImmediatePriority
2. UserBlockingPriority
3. NormalPriority
4. LowPriority
5. IdlePriority -> 当浏览器空闲时才执行
6. NoPriority (当没有指明以上priority时的priority)

// scheduler/src/Scheduler.js 各个优先级过期时间 -> 过期时则立马执行

`expirationTime = eventTime + timeout`

```javascript
var maxSigned31BitInt = 1073741823;
// Times out immediately
var IMMEDIATE_PRIORITY_TIMEOUT = -1;
// Eventually times out
var USER_BLOCKING_PRIORITY_TIMEOUT = 250;
var NORMAL_PRIORITY_TIMEOUT = 5000;
var LOW_PRIORITY_TIMEOUT = 10000;
// Never times out
var IDLE_PRIORITY_TIMEOUT = maxSigned31BitInt;
```

// packages/scheduler/src/SchedulerMinHeap.js

任务nodes通过最小堆来存储，优先级越高越靠前

### batchedEventUpdates, batchedUpdates
* packages/react-reconciler/src/ReactFiberReconciler.js
* packages/react-reconciler/src/ReactFiberWorkLoop.new.js and ReactFiberWorkLoop.old.js
* packages/react-dom/index.js
* packages/react-dom/src/client/ReactDOM.js


**ReactFiberReconciler中的batchedUpdates**
```javascript
// from ReactFiberReconciler
export function batchedUpdates<A, R>(fn: A => R, a: A): R {
  const prevExecutionContext = executionContext;
  executionContext |= BatchedContext; // 切换到批量更新上下文环境
  try {
    return fn(a);
  } finally {
    executionContext = prevExecutionContext; // 更新成功与否都回到之前都上下文环境
    if (executionContext === NoContext) {
      // Flush the immediate callbacks that were scheduled during this batch
      flushSyncCallbackQueue();
    }
  }
}
```

```javascript
// inside packages/react-dom/src/events/ReactDOMUpdateBatching.js

// Defaults 默认批量更新方法
let batchedUpdatesImpl = function(fn, bookkeeping) {
  return fn(bookkeeping);
};
let discreteUpdatesImpl = function(fn, a, b, c, d) {
  return fn(a, b, c, d);
};
let flushDiscreteUpdatesImpl = function() {};
let batchedEventUpdatesImpl = batchedUpdatesImpl;

export function
setBatchingImplementation(
  _batchedUpdatesImpl,
  _discreteUpdatesImpl,
  _flushDiscreteUpdatesImpl,
  _batchedEventUpdatesImpl,
) {
  batchedUpdatesImpl = _batchedUpdatesImpl;
  discreteUpdatesImpl = _discreteUpdatesImpl;
  flushDiscreteUpdatesImpl = _flushDiscreteUpdatesImpl;
  batchedEventUpdatesImpl = _batchedEventUpdatesImpl;
}
```

```javascript
// inside react-dom/src/client/ReactDOM.js

// 将react-reconciler/ReactFiberReconciler中的批量更新方法添加到ReactDOM.js中
setBatchingImplementation(
  batchedUpdates,
  discreteUpdates,
  flushDiscreteUpdates,
  batchedEventUpdates,
);

// 也向外通过unstable_batchedUpdates暴露了批量更新的方法
// ReactDOM.unstable_batchedUpdates(fn)
```

**react-dom/src/events/ReactDOMUpdateBatching.js**
```javascript
// 如果是在事件处理回调中的话，需要进行批量更新
export function batchedUpdates(fn, bookkeeping) {
  if (isInsideEventHandler) {
    // If we are currently inside another batch, we need to wait until it
    // fully completes before restoring state.
    return fn(bookkeeping);
  }
  isInsideEventHandler = true;
  try {
    return batchedUpdatesImpl(fn, bookkeeping);
  } finally {
    isInsideEventHandler = false;
    finishEventHandler();
  }
}

// 如果是别的批量更新中的批量更新的话，需要等待外层的更新完再批量更新
export function batchedEventUpdates(fn, a, b) {
  if (isBatchingEventUpdates) {
    // If we are currently inside another batch, we need to wait until it
    // fully completes before restoring state.
    return fn(a, b);
  }
  isBatchingEventUpdates = true;
  try {
    return batchedEventUpdatesImpl(fn, a, b);
  } finally {
    isBatchingEventUpdates = false;
    finishEventHandler();
  }
}

function finishEventHandler() {
  // Here we wait until all updates have propagated, which is important
  // when using controlled components within layers:
  // https://github.com/facebook/react/issues/1698
  // Then we restore state of any controlled component.
  const controlledComponentsHavePendingUpdates = needsStateRestore(); 
  // 当还有setState时，需要保持state不变，不更新到DOM上
  
  if (controlledComponentsHavePendingUpdates) {
    // If a controlled event was fired, we may need to restore the state of
    // the DOM node back to the controlled value. This is necessary when React
    // bails out of the update without touching the DOM.
    flushDiscreteUpdatesImpl(); // 如果有需要单独更新的则执行
    restoreStateIfNeeded();
    // 恢复未更新时的状态
    // 获取每个target的fiber instance
  }
}
```

#### react/src/ReactBaseClasses.js

```javascript
Component.prototype.setState = function(partialState, callback) {
  invariant(
    typeof partialState === 'object' ||
      typeof partialState === 'function' ||
      partialState == null,
    'setState(...): takes an object of state variables to update or a ' +
      'function which returns an object of state variables.',
  );
  this.updater.enqueueSetState(this, partialState, callback, 'setState');
};
```

#### react-reconciler/src/ReactFiberClassComponent.js
提供了基本的`fiber component updater`

```javascript
const classComponentUpdater = {
    ...
    // 将setState加入更新队列
    enqueueSetState(inst, payload, callback) { 
    const fiber = getInstance(inst);
    const eventTime = requestEventTime();
    const suspenseConfig = requestCurrentSuspenseConfig();
    const lane = requestUpdateLane(fiber, suspenseConfig);

    const update = createUpdate(eventTime, lane, suspenseConfig);
    update.payload = payload;
    if (callback !== undefined && callback !== null) {
      if (__DEV__) {
        warnOnInvalidCallback(callback, 'setState');
      }
      update.callback = callback;
    }

    enqueueUpdate(fiber, update); // 将新的update添加到fiber的updateQueue上
    scheduleUpdateOnFiber(fiber, lane, eventTime);

    if (__DEV__) {
      if (enableDebugTracing) {
        if (fiber.mode & DebugTracingMode) {
          const name = getComponentName(fiber.type) || 'Unknown';
          logStateUpdateScheduled(name, lane, payload);
        }
      }
    }

    if (enableSchedulingProfiler) {
      markStateUpdateScheduled(fiber, lane);
    }
  },
}
```

#### react-reconciler/src/ReactFiberBeginWork.new.js
`beginWork()` 根据不同的`Component Type`创建了不同连接`fiber`的`Component instance`并对`fiber`上的`work`进行执行然后返回下一个单位的`work`



#### react-reconciler/src/ReactFiberWorkLoop.new.js
当`ReactComponent`调用`setState`方法时，都会先将`updates`放进对应`fiber`当`updateQueue`中，然后调用`scheduleUpdateOnFiber`对更新进行调度。


```javascript
function scheduleUpdateOnFiber(
  fiber: Fiber,
  lane: Lane,
  eventTime: number,
) {
  checkForNestedUpdates(); // 检查是否有嵌套的更新 componentDidUpdate里有setState
  warnAboutRenderPhaseUpdatesInDEV(fiber);

  // 更新fiber上的lane，直到fiber root，将fiber root标记上有pending update
  const root = markUpdateLaneFromFiberToRoot(fiber, lane);
  if (root === null) {
    warnAboutUpdateOnUnmountedFiberInDEV(fiber);
    return null;
  }
  // TODO: requestUpdateLanePriority also reads the priority. Pass the
  // priority as an argument to that function and this one.
  const priorityLevel = getCurrentPriorityLevel(); // 获取当前优先级

  if (lane === SyncLane) { // 同步
    if (
      // 在非批量更新上下文或者非渲染和提交上下文的情况下
      // Check if we're inside unbatchedUpdates
      (executionContext & LegacyUnbatchedContext) !== NoContext &&
      // Check if we're not already rendering
      (executionContext & (RenderContext | CommitContext)) === NoContext
    ) {
      // Register pending interactions on the root to avoid losing traced interaction data.
      // 在root上注册pending interactions避免丢失
      schedulePendingInteractions(root, lane); // 最终触发scheduleInteractions(root, lane, __interactionsRef.current)

      // This is a legacy edge case. The initial mount of a ReactDOM.render-ed
      // root inside of batchedUpdates should be synchronous, but layout updates
      // should be deferred until the end of the batch.
      // 在root上执行同步任务
      performSyncWorkOnRoot(root);
    } else {
    // 在root上安排一个任务，每个root只有一个task，如果已经有任务被安排了，要保证已经存在的任务的expiration time和下一级root需要工作的task的expiration time一致
      ensureRootIsScheduled(root, eventTime);
      schedulePendingInteractions(root, lane);
      if (executionContext === NoContext) {
        // Flush the synchronous work now, unless we're already working or inside
        // a batch. This is intentionally inside scheduleUpdateOnFiber instead of
        // scheduleCallbackForFiber to preserve the ability to schedule a callback
        // without immediately flushing it. We only do this for user-initiated
        // updates, to preserve historical behavior of legacy mode.
        flushSyncCallbackQueue();
      }
    }
  } else {
    // Schedule a discrete update but only if it's not Sync.
    if (
      (executionContext & DiscreteEventContext) !== NoContext &&
      // Only updates at user-blocking priority or greater are considered
      // discrete, even inside a discrete event.
      (priorityLevel === UserBlockingSchedulerPriority ||
        priorityLevel === ImmediateSchedulerPriority)
    ) {
      // This is the result of a discrete event. Track the lowest priority
      // discrete update per root so we can flush them early, if needed.
      if (rootsWithPendingDiscreteUpdates === null) {
        rootsWithPendingDiscreteUpdates = new Set([root]);
      } else {
        rootsWithPendingDiscreteUpdates.add(root);
      }
    }
    // Schedule other updates after in case the callback is sync.
    ensureRootIsScheduled(root, eventTime);
    schedulePendingInteractions(root, lane);
  }
  // We use this when assigning a lane for a transition inside
  // `requestUpdateLane`. We assume it's the same as the root being updated,
  // since in the common case of a single root app it probably is. If it's not
  // the same root, then it's not a huge deal, we just might batch more stuff
  // together more than necessary.
  mostRecentlyUpdatedRoot = root;
}

```
---
`ensureRootIsScheduled 源码`

* `ensureRootIsScheduled`保证了`root`上的`task`被调度安排好了，如果之前`root`上有`pending task`，会重新设定它的`expiration time`和下一级`task`的`expiration time`一样（延迟更新 batchedUpdates)。
* 取消了`existingCallbackNode`上的`callback`，并安排新的`callback`。
* [ReactFiberLane源码](https://github.com/facebook/react/blob/master/packages/react-reconciler/src/ReactFiberLane.js) `lanes`是一个二进制序列，当某条`lane`到了过期时限时，会通过`binary |`添加到`root.expirationLanes`上，
    * `0b0000000000000000000000000000000` -> `0b0000000000000000000000000000001`，index为0的lane标记为过期了

```javascript
// Use this function to schedule a task for a root. There's only one task per
// root; if a task was already scheduled, we'll check to make sure the
// expiration time of the existing task is the same as the expiration time of
// the next level that the root has work on. This function is called on every
// update, and right before exiting a task.
function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
  const existingCallbackNode = root.callbackNode;

  // Check if any lanes are being starved by other work. If so, mark them as
  // expired so we know to work on those next.
  // 检查有没有已经过期的任务，有则加到root.expiredLanes上
  markStarvedLanesAsExpired(root, currentTime);

  // Determine the next lanes to work on, and their priority.
  // 判断接下来要执行的lane，任务和它们的优先级
  const newCallbackId = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes,
  );
  // This returns the priority level computed during the `getNextLanes` call.
  // 计算新任务的优先级
  const newCallbackPriority = returnNextLanesPriority();

  if (newCallbackId === NoLanes) {
    // Special case: There's nothing to work on.
    if (existingCallbackNode !== null) {
      cancelCallback(existingCallbackNode); 
      root.callbackNode = null;
      root.callbackPriority = NoLanePriority;
      root.callbackId = NoLanes;
    }
    return;
  }

  // 检查有没有已存在的任务
  // Check if there's an existing task. We may be able to reuse it.
  const existingCallbackId = root.callbackId;
  const existingCallbackPriority = root.callbackPriority;
  if (existingCallbackId !== NoLanes) { // 有existing task
    if (newCallbackId === existingCallbackId) {
      // 新task已经被安排了
      // This task is already scheduled. Let's check its priority.
      if (existingCallbackPriority === newCallbackPriority) {
        // The priority hasn't changed. Exit.
        // 检查优先级有没有变化，没有则退出
        return;
      }
      // The task ID is the same but the priority changed. Cancel the existing
      // callback. We'll schedule a new one below.
      // 如果任务id没变但是优先级变了，先取消之前的回调
      // 再安排一个新的
    }
    // 取消之前的task callback
    cancelCallback(existingCallbackNode);
  }

  // 安排新的 task callback
  // Schedule a new callback.
  let newCallbackNode;
  if (newCallbackPriority === SyncLanePriority) { 
    // 同步任务
    // 被安排在一个特殊的内部队列里
    // Special case: Sync React callbacks are scheduled on a special
    // internal queue
    newCallbackNode = scheduleSyncCallback(
      performSyncWorkOnRoot.bind(null, root),
    );
  } else {
    // 普通异步任务，批量更新任务
    // 获取新的调度优先级
    const schedulerPriorityLevel = lanePriorityToSchedulerPriority(
      newCallbackPriority,
    );
    // 新的任务批量任务
    newCallbackNode = scheduleCallback(
      schedulerPriorityLevel,
      performConcurrentWorkOnRoot.bind(null, root), // callback为执行同时更新任务
    );
  }
  // 给当前root安排新的callbackId，优先级和节点
  root.callbackId = newCallbackId;
  root.callbackPriority = newCallbackPriority;
  root.callbackNode = newCallbackNode;
}
```
---
`scheduler/src/Scheduler `

* `unstable_scheduleCallback`是`ensureRootIsScheduled`中的`scheduleCallback`，规划返回了新的任务。
* `requestHostCallback`通过`postMessage`发起`performWorkUntilDeadline`，在deadline前执行更新操作，超过deadline将主线程控制交还给浏览器进行渲染和用户输入响应。
* `flushWork`返回`workLoop`

```javascript
 function unstable_scheduleCallback(priorityLevel, callback, options) {
  var currentTime = getCurrentTime();

  var startTime;
  var timeout;
  if (typeof options === 'object' && options !== null) {
    var delay = options.delay;
    if (typeof delay === 'number' && delay > 0) {
      startTime = currentTime + delay;
    } else {
      startTime = currentTime;
    }
    timeout =
      typeof options.timeout === 'number'
        ? options.timeout
        : timeoutForPriorityLevel(priorityLevel);
  } else {
    timeout = timeoutForPriorityLevel(priorityLevel);
    startTime = currentTime;
  }

  var expirationTime = startTime + timeout;

  var newTask = {
    id: taskIdCounter++,
    callback,
    priorityLevel,
    startTime,
    expirationTime,
    sortIndex: -1,
  };
  if (enableProfiling) {
    newTask.isQueued = false;
  }

  if (startTime > currentTime) { // 延迟任务
    // This is a delayed task.
    newTask.sortIndex = startTime;
    // timerQueue是一个以startTime排序的最小堆，延迟任务用
    push(timerQueue, newTask);
    // peek返回堆中的第一个node
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      // All tasks are delayed, and this is the task with the earliest delay.
      // 如果所有任务都是延迟任务，并且新加的任务是延迟最短的
      if (isHostTimeoutScheduled) {
        // Cancel an existing timeout.
        // 若有新的延迟更短的任务，取消之前的
        cancelHostTimeout();
      } else {
        isHostTimeoutScheduled = true;
      }
      // Schedule a timeout.
      // 安排一个timeout
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    // 非延迟任务，加入taskQueue，以expirationTime排序的min heap
    newTask.sortIndex = expirationTime;
    push(taskQueue, newTask);
    if (enableProfiling) {
      markTaskStart(newTask, currentTime);
      newTask.isQueued = true;
    }
    // Schedule a host callback, if needed. If we're already performing work,
    // wait until the next time we yield.
    // 安排一个浏览器回调，如果已经有正在进行的更
    // 新工作，则留到下一次变更控制权的时候再安排
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      requestHostCallback(flushWork);
      // flushWork会被设为requestHostCallback中的scheduledHostCallback
      // flushWork在Scheduler.js中
    }
  }

  return newTask;
}   
```

---

`workLoop(hasTimeRemaining, initialTime) Scheduler.js`
* `markTask***`起到标记作用，指明task在workloop中进行到哪一步了

```javascript
function workLoop(hasTimeRemaining, initialTime) {
  let currentTime = initialTime;
  advanceTimers(currentTime);
  currentTask = peek(taskQueue);
  while (
    currentTask !== null &&
    !(enableSchedulerDebugging && isSchedulerPaused)
  ) {
    if (
      currentTask.expirationTime > currentTime &&
      (!hasTimeRemaining || shouldYieldToHost())
    ) {
      // This currentTask hasn't expired, and we've reached the deadline.
      // 当前任务还没有到期但是分帧空闲时间已经没有了
      break;
    }
    const callback = currentTask.callback;
    if (callback !== null) {
      currentTask.callback = null;
      currentPriorityLevel = currentTask.priorityLevel;
      const didUserCallbackTimeout = currentTask.expirationTime <= currentTime;
      markTaskRun(currentTask, currentTime);
      const continuationCallback = callback(didUserCallbackTimeout); 
      // callback为 
      // performConcurrentWorkOnRoot(root, didTimeout)
      // 如果已经过期
      currentTime = getCurrentTime();
      if (typeof continuationCallback === 'function') {
        currentTask.callback = continuationCallback;
        markTaskYield(currentTask, currentTime);
      } else {
        if (enableProfiling) {
          markTaskCompleted(currentTask, currentTime);
          currentTask.isQueued = false;
        }
        if (currentTask === peek(taskQueue)) {
          pop(taskQueue);
        }
      }
      advanceTimers(currentTime);
    } else {
      pop(taskQueue);
    }
    currentTask = peek(taskQueue); // 抽取下一个任务
  }
  // Return whether there's additional work
  // 还有更多的任务，但是scheduler被暂停了
  // 会再次postMessage触发performWorkUntilDeadline
  // 留给接下来的scheduleHostCallback
  if (currentTask !== null) {
    return true;
  } else {
    const firstTimer = peek(timerQueue);
    if (firstTimer !== null) {
      requestHostTimeout(handleTimeout, firstTimer.startTime - currentTime);
    }
    return false;
  }
}
```