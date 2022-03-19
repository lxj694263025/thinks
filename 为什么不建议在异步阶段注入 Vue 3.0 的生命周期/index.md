# 为什么不建议在异步阶段注入 Vue 3.0 的生命周期

![](assets/cover.jpg)

## 前言

我们在使用 Vue 3.0 Composition API 时，通常会在 `setup` 周期中利用生命周期 hooks 函数（`onMounted`、`onBeforeDestroy` 等）完成对生命周期钩子的注入。那调用这些 API 时有没有一些限制呢，答案是肯定的，我们先通过例子看一下现象。

## 不同阶段注入生命周期钩子对比

### 同步阶段

这是我们通常书写生命周期注入的方式，当该组件加载时，控制台能正常打印出 `mounted`。

```vue
<template>
  <div />
</template>

<script lang="ts">
import { onMounted } from 'vue';

export default {
  setup() {
    onMounted(() => {
      console.log('mounted');
    });
  },
};
</script>
```

### 异步阶段

如果我们突发奇想想要在异步阶段注入生命周期会发生什么现象？

```javascript
export default {
  setup() {
    setTimeout(() => {
      onMounted(() => {
        console.log('mounted');
      });
    });
  },
};
```

此时我们会发现，控制台输出了一个 Vue 的警告：`[Vue warn]: onMounted is called when there is no active component instance to be associated with. Lifecycle injection APIs can only be used during execution of setup(). If you are using async setup(), make sure to register lifecycle hooks before the first await statement.`。

大概意思就是，`onMounted` 被调用时，当前并没有活跃状态的组件实例去处理生命周期钩子的注入。生命周期钩子的注入只能在 `setup` 同步执行期间进行，如果我们想要在 `async` 形态的异步 `setup` 中注入生命周期钩子，必须确保在第一个 `await` 之前进行。

## 从源码看现象

### 2.0 中的依赖收集和派发更新

Vue 2.0 在进行依赖收集时会将当前正在创建的组件实例 `Watcher` 存入到 `Dep.target` 这个变量中，这个做法就可以很容易地将当前组件实例和组件需要的变量依赖关联起来。组件在创建时需要读取一些变量，这些变量是经过 `defineReactive` 封装过的，其中存在一个 `Dep` 实例用来维护所有依赖该变量的 `Watcher`。当某个变量被读取时，这个变量的 `getter` 拦截器就会向属于它的 `Dep` 实例中注册当前正在创建的组件实例 `Dep.target`。当该变量更新时，变量的 `setter` 拦截器就会遍历 `dep.subs` 队列并通知每一个 `Watcher` 进行 `update` 更新。我简单地描述了一下 Vue 2.0 中的依赖收集和派发更新的过程，其中一个关键的步骤就是将当前正在创建的组件点亮到全局标记 `Dep.target` 上使得依赖变量能收集到它的被依赖者。

有了这个前提，那我大胆猜想一下，Vue 3.0 处理生命周期 Composition API 时也是借助了类似的思想，将当前正在创建的组件实例点亮到全局标记上来完成 hooks 和它所处组件的正确关联。下面我们就通过 3.0 源码来验证一下我的猜想是否正确。

### 3.0 中对生命周期 hooks 的定义

我在 3.0 的 [`packages/runtime-core/src/apiLifecycle.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/apiLifecycle.ts#L73) 中找到了对生命周期 hooks 函数的定义。

```typescript
export const onBeforeMount = createHook(LifecycleHooks.BEFORE_MOUNT)
export const onMounted = createHook(LifecycleHooks.MOUNTED)
export const onBeforeUpdate = createHook(LifecycleHooks.BEFORE_UPDATE)
export const onUpdated = createHook(LifecycleHooks.UPDATED)
export const onBeforeUnmount = createHook(LifecycleHooks.BEFORE_UNMOUNT)
export const onUnmounted = createHook(LifecycleHooks.UNMOUNTED)
export const onServerPrefetch = createHook(LifecycleHooks.SERVER_PREFETCH)
export const onRenderTriggered = createHook<DebuggerHook>(
  LifecycleHooks.RENDER_TRIGGERED
)
export const onRenderTracked = createHook<DebuggerHook>(
  LifecycleHooks.RENDER_TRACKED
)
```

这些 hooks 函数是由 [`createHook`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/apiLifecycle.ts#L66) 函数创建出来的新函数，都传入了 `LifecycleHooks` 枚举中的值来标明自己的身份，我们再来看看 `createHook` 函数做了什么。

```typescript
export const createHook =
  <T extends Function = () => any>(lifecycle: LifecycleHooks) =>
  (hook: T, target: ComponentInternalInstance | null = currentInstance) =>
    // post-create lifecycle registrations are noops during SSR (except for serverPrefetch)
    (!isInSSRComponentSetup || lifecycle === LifecycleHooks.SERVER_PREFETCH) &&
    injectHook(lifecycle, hook, target)
```

我们仔细看发现它直接返回了一个新函数，我们在业务代码中调用 `onMounted` 这样的钩子时，就会执行这个函数，它接收一个 `hook` 回调函数，以及一个非必填的 `target` 对象，默认值是 `currentInstance`，我们稍后分析这个 `target` 对象是什么。我们先不考虑 SSR 的情况，最终执行的 `injectHook` 是关键操作，看字面意思就是将 `hook` 回调函数注入到 `target` 对象的指定生命周期 `lifecycle` 中。

我们继续挖掘 [`injectHook`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/apiLifecycle.ts#L17) 做了什么，我刨去了不重要的部分。

```typescript
export function injectHook(
  type: LifecycleHooks,
  hook: Function & { __weh?: Function },
  target: ComponentInternalInstance | null = currentInstance,
  prepend: boolean = false
): Function | undefined {
  if (target) {
    // 先根据指定的 lifecycle type 从 target 中寻找对应的 hooks 队列
    const hooks = target[type] || (target[type] = [])
    // 省略
    // 这是经过包装的 hook 执行函数，用于在实际调用 hook 钩子时处理边界、错误等情况
    const wrappedHook = /* 省略 */ (...args: unknown[]) => {
      // 省略
      const res = callWithAsyncErrorHandling(hook, target, type, args)
      // 省略
      return res
    }
    // 下面是关键步骤，我们发现它将 hook 回调函数注入到了对应生命周期的 hooks 队列中
    if (prepend) {
      hooks.unshift(wrappedHook)
    } else {
      hooks.push(wrappedHook)
    }
    return wrappedHook
  } else if (__DEV__) {
    // 省略
  }
}
```

我们可以看到 `injectHook` 确实是将 hook 回调钩子注入到 `target` 对应的生命周期队列中。这就是 `onMounted` 调用后发生的效果。

### `target` 是什么

上面我们保留了一个疑问，就是 `target` 到底是什么，我们很容易就能根据它的类型描述 `ComponentInternalInstance` 以及默认值 `currentInstance` 联想到它是当前正在创建组件的实例。我们继续验证我们的猜想。

我通过 `currentInstance` 的引入位置 [`packages/runtime-core/src/component.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/component.ts#L550) 找到了它的一系列设置函数。

```typescript
export let currentInstance: ComponentInternalInstance | null = null

export const getCurrentInstance: () => ComponentInternalInstance | null = () =>
  currentInstance || currentRenderingInstance

export const setCurrentInstance = (instance: ComponentInternalInstance) => {
  currentInstance = instance
  instance.scope.on()
}

export const unsetCurrentInstance = () => {
  currentInstance && currentInstance.scope.off()
  currentInstance = null
}
```

我们继续找 `setCurrentInstance` 的调用源头，我在同文件中发现 [`setupStatefulComponent`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/component.ts#L600) 函数调用了它，

```typescript
function setupStatefulComponent(
  instance: ComponentInternalInstance,
  isSSR: boolean
) {
  const Component = instance.type as ComponentOptions
  // 省略
  // 0. create render proxy property access cache
  instance.accessCache = Object.create(null)
  // 1. create public instance / render proxy
  // also mark it raw so it's never observed
  // 省略
  // 2. call setup()
  const { setup } = Component
  if (setup) {
    // 创建 setup 函数上下文入参
    const setupContext = (instance.setupContext =
      setup.length > 1 ? createSetupContext(instance) : null)

    // 关键步骤，点亮当前组件实例，必须在 setup 函数被调用前
    setCurrentInstance(instance)
    pauseTracking()
    // 调用 setup 函数
    const setupResult = callWithErrorHandling(
      setup,
      instance,
      ErrorCodes.SETUP_FUNCTION,
      [__DEV__ ? shallowReadonly(instance.props) : instance.props, setupContext]
    )
    resetTracking()
    // 关键步骤，解除当前点亮组件的设置
    unsetCurrentInstance()
    // 省略
  } else {
    // 省略
  }
}
```

同样刨去不重要的逻辑，我们可以看到关键的逻辑中，确实做了在 `setup` 函数调用前点亮当前正在创建组件的实例这样的操作，但是到目前为止还是不能明确地说明这个 `instance`(`target`) 就是组件的实例。我们继续向上查找，找到了一个叫 [`mountComponent`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/renderer.ts#L1186) 的函数，在其中进行了组件实例的创建和 `setupComponent` 的调用。

```typescript
const mountComponent: MountComponentFn = (
  initialVNode,
  container,
  anchor,
  parentComponent,
  parentSuspense,
  isSVG,
  optimized
) => {
  // 2.x compat may pre-create the component instance before actually
  // mounting
  const compatMountInstance =
    __COMPAT__ && initialVNode.isCompatRoot && initialVNode.component
  const instance: ComponentInternalInstance =
    compatMountInstance ||
    (initialVNode.component = createComponentInstance(
      initialVNode,
      parentComponent,
      parentSuspense
    ))
  // 省略
  // resolve props and slots for setup context
  if (!(__COMPAT__ && compatMountInstance)) {
    // 省略
    setupComponent(instance)
    // 省略
  }
  // 省略
}
```

至此，我们知道 `target` 确实是当前正在创建的组件实例，并且了解到了生命周期 hooks 和所在组件实例之间发生关联的整个过程，猜想已全部验证。

## 更多的思考

### 明确不能在异步阶段调用生命周期钩子的原因

因为 Vue 在调用 `setup` 函数时是非阻塞式的，这意味着 `setup` 函数同步执行周期结束之后，Vue 就立马解除了当前点亮组件的设置，这就很容易理解为什么 Vue 对异步生命周期 hooks 的注入发出了警告。

```typescript
setCurrentInstance(instance)
// 省略
const setupResult = callWithErrorHandling(
  setup,
  instance,
  ErrorCodes.SETUP_FUNCTION,
  [__DEV__ ? shallowReadonly(instance.props) : instance.props, setupContext]
)
// 省略
unsetCurrentInstance()
```

### 如何在异步阶段注入生命周期

根据 Vue 的建议，推荐我们在 `setup` 同步执行周期内注入生命周期。除此之外，我们通过观察 `createHook` 函数所创建出的函数的入参（`hook: T, target: ComponentInternalInstance | null = currentInstance`）发现它允许我们手动传一个组件实例进行生命周期的注入。

有了这种特性再结合 Vue 官方提供了 `getCurrentInstance` 函数用于获取当前组件实例，我再大胆地猜想一下，我们可以异步地注入组件销毁阶段的生命周期，接下来我们来尝试一下。

```vue
<!-- parent.vue -->
<template>
  <div>
    <async-lifecycle v-if="isShow" />
    <button @click="hide">hide</button>
  </div>
</template>

<script lang="ts">
import { ref } from 'vue';
import AsyncLifecycle from './async-lifecycle.vue';

export default {
  components: {AsyncLifecycle},
  setup() {
    const isShow = ref(true);
    const hide = () => {
      isShow.value = false;
    };
    return {
      isShow,
      hide,
    };
  },
};
</script>
```

我们先定义一个父组件，在其中引入了一个子组件 `async-lifecycle`，由父组件控制其显隐，默认处于显示的状态。子组件的定义如下：

```vue
<!-- async-lifecycle.vue -->
<template>
  <div />
</template>

<script lang="ts">
import { getCurrentInstance, onUnmounted } from 'vue';

export default {
  setup() {
    // getCurrentInstance 函数也必须在 setup 同步周期内调用
    const instance = getCurrentInstance();
    setTimeout(() => {
      // 异步注入组件卸载时的生命周期
      onUnmounted(() => {
        console.log('unmounted');
      }, instance);
    });
  },
}
</script>
```

我们点击父组件的按钮对子组件进行隐藏时，发现控制台输出了 `unmounted`，猜想成功！

## 小结

我们通过 Vue 源码了解到了生命周期类的 Composition API 与组件实例的关联方式，一些和当前组件有关联的其它类型 API 也是类似的思路，感兴趣的同学可以去了解一下它们的实现原理。有了这些认知之后我们书写 Composition API 时应当小心，尽可能避免异步的调用，如果调试时发现了类似问题也可以按这种思路顺藤摸瓜寻找不恰当的调用时机。

最后推荐一个学习 Vue 3.0 实现原理的捷径 [mini-vue](https://github.com/cuixiaorui/mini-vue) 。

> 当我们需要深入学习 vue3 时，我们就需要看源码来学习，但是像这种工业级别的库，源码中有很多逻辑是用于处理边缘情况或者是兼容处理逻辑，是不利于我们学习的。
>
> 我们应该关注于核心逻辑，而这个库的目的就是把 vue3 源码中最核心的逻辑剥离出来，只留下核心逻辑，以供大家学习。
