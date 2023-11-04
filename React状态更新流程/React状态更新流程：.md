# React状态更新流程：

## 在react中触发状态更新的几种方式：

- ReactDOM.render
- this.setState
- this.forceUpdate
- useState
- useReducer

我们重点看下重点看下this.setState和this.forceUpdate：

## 1. setState:

setState主要是调用了enqueueSetState，将update塞入了updateQueue中。

```js
// packages/react/src/ReactBaseClasses.js
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

### 1.1 enqueueSetState:

enqueueSetState中创建了update对象，并将其塞入updateQueue中。

```js
enqueueSetState(inst, payload, callback) {
  const fiber = getInstance(inst); // 找到fiber对象
  const eventTime = requestEventTime();
  const lane = requestUpdateLane(fiber);

  const update = createUpdate(eventTime, lane); // 创建update对象
  update.payload = payload;
  if (callback !== undefined && callback !== null) {
    update.callback = callback;
  }
	
  enqueueUpdate(fiber, update); // 加入到updateQueue中
  scheduleUpdateOnFiber(fiber, lane, eventTime); //调度update

  if (enableSchedulingProfiler) {
    markStateUpdateScheduled(fiber, lane);
  }
}
```

#### enqueueUpdate:

```js
export function enqueueUpdate<State>(fiber: Fiber, update: Update<State>) {
  const updateQueue = fiber.updateQueue;
  if (updateQueue === null) {
    return;
  }

  const sharedQueue: SharedQueue<State> = (updateQueue: any).shared;
  const pending = sharedQueue.pending;
  if (pending === null) {
    // This is the first update. Create a circular list.
    update.next = update; // 如果是第一个update，加入的update next指向自己形成自循环
  } else {
    update.next = pending.next; // 加入updateQueue链表结尾
    pending.next = update;
  }
  sharedQueue.pending = update; // 更新update队列
}
```

## 2. forceUpdate:

```js
Component.prototype.forceUpdate = function(callback) {
  this.updater.enqueueForceUpdate(this, callback, 'forceUpdate');
};
```

### 2.1 enqueueForceUpdate:

enqueueForceUpdate逻辑和enqueueSetState差不多，都是创建update对象，加入updateQueue中，不过会对update打上tag ForceUpdate

```js
enqueueForceUpdate(inst, callback) {
  const fiber = getInstance(inst);
  const eventTime = requestEventTime();
  const lane = requestUpdateLane(fiber);

  const update = createUpdate(eventTime, lane);
  update.tag = ForceUpdate;	// forceUpdate 打上tag

  if (callback !== undefined && callback !== null) {
    update.callback = callback;
  }

  if (enableSchedulingProfiler) {
    markForceUpdateScheduled(fiber, lane);
  }
},
```

 如果标记ForceUpdate，render阶段组件更新会根据checkHasForceUpdateAfterProcessing，和checkShouldComponentUpdate来判断

- 如果Update的tag是ForceUpdate，则checkHasForceUpdateAfterProcessing为true

- 当组件是PureComponent时，checkShouldComponentUpdate会浅比较state和props

```js
const shouldUpdate =
  checkHasForceUpdateAfterProcessing() ||
  checkShouldComponentUpdate(
    workInProgress,
    ctor,
    oldProps,
    newProps,
    oldState,
    newState,
    nextContext,
  );
```

## 状态更新整体流程：

![状态更新整体流程](/Users/bk/Desktop/学习资料/react源码学习/React状态更新流程/状态更新整体流程.png)

## Update&updateQueue：

HostRoot或者ClassComponent触发更新后，会在函数createUpdate中创建update，并在后面的**render阶段的beginWork中计算Update**。

### createUpdate：

```js
export function createUpdate(eventTime: number, lane: Lane): Update<*> {
  const update: Update<*> = {
    eventTime,	
    lane,	// 优先级

    tag: UpdateState,	// tag 更新类型
    payload: null,	// 本次更新state
    callback: null,	// 回调函数

    next: null,	// 下一个update对象
  };
  return update;
}
```

**我们主要关注这些参数：**

- lane：优先级
- tag：更新的类型，例如UpdateState、ReplaceState
- payload：ClassComponent的payload是setState第一个参数，HostRoot的payload是ReactDOM.render的第一个参数
- callback：setState的第二个参数
- next：连接下一个Update形成一个链表，例如同时触发多个setState时会形成多个Update，然后用next 连接

### initializeUpdateQueue：

对于HostRoot或者ClassComponent会在mount的时候使用initializeUpdateQueue创建updateQueue，然后将updateQueue挂载到fiber节点上

```
export function initializeUpdateQueue<State>(fiber: Fiber): void {
  const queue: UpdateQueue<State> = {
    baseState: fiber.memoizedState,
    firstBaseUpdate: null,
    lastBaseUpdate: null,
    shared: {
      pending: null,
    },
    effects: null,
  };
  fiber.updateQueue = queue;
}
```

- baseState：初始state，后面会基于这个state，根据Update计算新的state
- firstBaseUpdate、lastBaseUpdate：Update形成的链表的头和尾
- shared.pending：**新产生的update会以单向环状链表保存在shared.pending**上，计算state的时候会剪开这个环状链表，并且链接在lastBaseUpdate后
- effects：calback不为null的update

### markUpdateLaneFromFiberToRoot：

**从触发更新的fiber节点向上遍历到rootFiber**

在markUpdateLaneFromFiberToRoot函数中，会从触发更新的节点向上遍历找到根节点FiberRootNode，遍历过程中会处理优先级。

```js
function markUpdateLaneFromFiberToRoot(
  sourceFiber: Fiber,
  lane: Lane,
): FiberRoot | null {
  sourceFiber.lanes = mergeLanes(sourceFiber.lanes, lane); // 更新优先级
  let alternate = sourceFiber.alternate;
  if (alternate !== null) {
    alternate.lanes = mergeLanes(alternate.lanes, lane);
  }
  let node = sourceFiber;
  let parent = sourceFiber.return;
  while (parent !== null) { // 循环遍历寻找rootFiber
    parent.childLanes = mergeLanes(parent.childLanes, lane); // 更新优先级
    alternate = parent.alternate;
    if (alternate !== null) {
      alternate.childLanes = mergeLanes(alternate.childLanes, lane);
    } else {
      ...
    }
    node = parent;
    parent = parent.return;
  }
  if (node.tag === HostRoot) {
    const root: FiberRoot = node.stateNode;
    return root;
  } else {
    return null;
  }
}
```

例如B节点触发更新，B节点被被标记为normal的update，也就是图中的u1，然后向上遍历到根节点，在根节点上打上一个normal的update，如果此时B节点又触发了一个userBlocking的Update，同样会向上遍历到根节点，在根节点上打上一个userBlocking的update。

如果当前根节点更新的优先级是normal，u1、u2都参与状态的计算，如果当前根节点更新的优先级是userBlocking，则只有u2参与计算

![状态更新1](/Users/bk/Desktop/学习资料/react源码学习/React状态更新流程/状态更新1.jpg)

### ensureRootIsScheduled调度：

在ensureRootIsScheduled中，scheduleCallback会以一个优先级调度render阶段的开始函数performSyncWorkOnRoot或者performConcurrentWorkOnRoot。

```js
if (newCallbackPriority === SyncLanePriority) {
 	// 任务已经过期，需要同步执行render阶段
  newCallbackNode = scheduleSyncCallback(
    performSyncWorkOnRoot.bind(null, root),
  );
} else{
  // 根据任务优先级异步执行render阶段
  const schedulerPriorityLevel = lanePriorityToSchedulerPriority(
    newCallbackPriority,
  );
  newCallbackNode = scheduleCallback(
    schedulerPriorityLevel,
    performConcurrentWorkOnRoot.bind(null, root),
  );
}
```

### processUpdateQueue状态更新：

在 `react` 的**更新流程**中，如果是 `ClassComponent（updateClassInstance函数）` 或者 `HostRoot（updateHostRoot函数）`，会走到 `processUpdateQueue` 函数内。

```js
export function processUpdateQueue<State>(
  workInProgress: Fiber,
  props: any,
  instance: any,
  renderLanes: Lanes,
): void {
  const queue: UpdateQueue<State> = (workInProgress.updateQueue: any);

  hasForceUpdate = false;

  if (__DEV__) {
    currentlyProcessingQueue = queue.shared;
  }

  let firstBaseUpdate = queue.firstBaseUpdate; // 取队列的第一个update firstBaseUpdate
  let lastBaseUpdate = queue.lastBaseUpdate; // 取队列的最后一个update lastBaseUpdate

  let pendingQueue = queue.shared.pending; // 取更新队列
  if (pendingQueue !== null) {
    // 如果更新队列不为空，将队列置空
    queue.shared.pending = null;

    // 记录第一个和最后一个 Update，同时把环形链表的环拆开
    const lastPendingUpdate = pendingQueue;
    const firstPendingUpdate = lastPendingUpdate.next;
    lastPendingUpdate.next = null;
    if (lastBaseUpdate === null) {
      // 如果 lastBaseUpdate 为空，firstBaseUpdate 和 lastBaseUpdate 就被赋值为 queue 的起点和终点
      firstBaseUpdate = firstPendingUpdate;
    } else {
      // 如果 lastBaseUpdate 不为空，那么会拼接到 queue 的前面
      lastBaseUpdate.next = firstPendingUpdate;
    }
    lastBaseUpdate = lastPendingUpdate;
    // 上述的操作其实是把 queue 环形队列解环
    // 然后把 firstBaseUpdate 和 lastBaseUpdate 构成的队列拼接到 queue 前面
    // 最后构成一个大的线性 UpdateQueue
    // firstBaseUpdate 和 lastBaseUpdate 分别作为队列的起点和终点

    const current = workInProgress.alternate;
    if (current !== null) {
      // 执行类似上述的操作，把 workInProgress 节点中的 queue 同步到 current 节点
      ...
    }
  }

  if (firstBaseUpdate !== null) {
    // 遍历updateQueue来计算最终更新的state
    // 新的state
    let newState = queue.baseState;
    
    // 新的优先级
    let newLanes = NoLanes;

    let newBaseState = null;
    let newFirstBaseUpdate = null;
    let newLastBaseUpdate = null;
    // 第一个update
    let update = firstBaseUpdate;
    // 队列循环处理
    do {
      const updateLane = update.lane;
      const updateEventTime = update.eventTime;
      if (!isSubsetOfLanes(renderLanes, updateLane)) {
        // 这里通过 lane 的工具函数判断出当前 update 的 lane 不满足更新优先级条件
        // 下面是把不满足更新条件的 update 用链表存了起来
        const clone: Update<State> = {
          eventTime: updateEventTime,
          lane: updateLane,

          tag: update.tag,
          payload: update.payload,
          callback: update.callback,

          next: null,
        };
        // newFirstBaseUpdate 和 newLastBaseUpdate 是队列的起点和终点
        if (newLastBaseUpdate === null) {
          // 这里有个很特殊的处理，如果出现有update被延迟执行，
          // 那么会把当前已经计算好的 newState 先做一次保存
          newFirstBaseUpdate = newLastBaseUpdate = clone;
          newBaseState = newState;
        } else {
          newLastBaseUpdate = newLastBaseUpdate.next = clone;
        }
        // 往 Update 对象的 Lanes 中塞入 updateLane，最后的newLanes会被塞给workInProgress.lanes
        newLanes = mergeLanes(newLanes, updateLane);
      } else {
        // 满足更新条件的 update
        if (newLastBaseUpdate !== null) {
          const clone: Update<State> = {
            eventTime: updateEventTime,
            // 这里通过设置 NoLane 保证了下一次检查时一定会通过上面的 isSubsetOfLanes
            lane: NoLane,

            tag: update.tag,
            payload: update.payload,
            callback: update.callback,

            next: null,
          };
          newLastBaseUpdate = newLastBaseUpdate.next = clone;
        }
        // 计算新的 state
        newState = getStateFromUpdate(
          workInProgress,
          queue,
          update,
          newState,
          props,
          instance,
        );
        // 保存 setState 的 callback，就是第二个参数
        const callback = update.callback;
        if (callback !== null) {
          // 标记当前 fiber 有 callback
          workInProgress.flags |= Callback;
          const effects = queue.effects;
          if (effects === null) {
            queue.effects = [update];
          } else {
            effects.push(update);
          }
        }
      }
      update = update.next;
      if (update === null) {
        pendingQueue = queue.shared.pending;
        if (pendingQueue === null) {
          break;
        } else {
          // 当前的 queue 处理完后，需要检查一下 queue.shared.pending 是否有更新
          // 然后把剩余的 update 都推入到 lastBaseUpdate
          // ... 这里涉及到链表拆环的代码，与上面的拆环没有太大差异
          const lastPendingUpdate = pendingQueue;
          const firstPendingUpdate = ((lastPendingUpdate.next: any): Update<State>);
          lastPendingUpdate.next = null;
          update = firstPendingUpdate;
          queue.lastBaseUpdate = lastPendingUpdate;
          queue.shared.pending = null;
        }
      }
    } while (true);

    if (newLastBaseUpdate === null) {
      // 这里有一个很关键的判断，只有在没出现update被延迟的情况下
      // 才会把整个队列的计算结果赋值给 newBaseState
      newBaseState = newState;
    }

    // newBaseState 就是新的 baseState，也就是后面界面展示时用到的 fiber.memoizedState
    queue.baseState = ((newBaseState: any): State);
    // 保存被延迟的 update
    queue.firstBaseUpdate = newFirstBaseUpdate;
    queue.lastBaseUpdate = newLastBaseUpdate;

    // 标记哪些区间的update被延迟了
    markSkippedUpdateLanes(newLanes);
    workInProgress.lanes = newLanes;
    // 只更新 workInProgress（新节点）的 memoizedState
    workInProgress.memoizedState = newState;
  }
}
```

首先processUpdateQueue会去获取到当前baseQueue的firstBaseUpdate和lastBaseUpdate，然后将pending链表环解开，接到baseQueue后面，形成一个新的一级链表。如下图![updateQueue处理](/Users/bk/Desktop/学习资料/react源码学习/React状态更新流程/updateQueue处理.png)

之后会循环遍历这个一级链表，循环前会先创建newLanes， newBaseState，newFirstBaseUpdate，newLastBaseUpdate等临时变量。

- 首先回去判断当前update的优先级是否满足，不满足则将当前的update加入到延迟update队列中，如果首次出现不满足优先级的update的话，还会先记录一下newState
- 如果满足优先级，则会先去看是否存在延迟的update，如果存在则之后的update都将接到延迟update队列后面，并且lane设为Nolane，然后通过**getStateFromUpdate**计算新的state，并保存callback（setState的第二个参数）

- 然后将指向下一个update，在进行下一次遍历前，当update是空的时候说明当前queue已经处理完，会去判断**pendingQueue**是否为空，为空就直接结束循环，非空则会把剩余的 update 都接到队列后面。
- 循环结束后会进行一个**关键的判断**，只有当没有推迟的update的时候，才会把**newState赋值给newBaseState**
- 之后更新queue.baseState，也就是后面界面展示时用到的 fiber.memoizedState，保存被延迟的 update，调用markSkippedUpdateLanes标记哪些区间的update被延迟了，最后更新 workInProgress（新节点）的 memoizedState

#### getStateFromUpdate计算state：

在processUpdateQueue中我们会调用getStateFromUpdate来计算新的state，其会根据不同的update.tag来走不同的逻辑

并且我们知道setState既可以传入**对象**也可以传入**函数**，在getStateFromUpdate中也对其做出了判断

```js
function getStateFromUpdate<State>(
  workInProgress: Fiber,
  queue: UpdateQueue<State>,
  update: Update<State>,
  prevState: State,
  nextProps: any,
  instance: any,
): any {
  switch (update.tag) {
    case ReplaceState: 
      ...
    case CaptureUpdate: 
      ...
    // Intentional fallthrough
    case UpdateState: {
      const payload = update.payload;
      let partialState;
      // setState可能传入的是函数
      if (typeof payload === 'function') {
        ...
        partialState = payload.call(instance, prevState, nextProps);
        ...
      } else {
        partialState = payload;
      }
      if (partialState === null || partialState === undefined) {
        return prevState;
      }
      // 用Object.assign实现旧值替换，增量更新
      return Object.assign({}, prevState, partialState);
    }
    case ForceUpdate: 
      ...
  }
  return prevState;
}
```