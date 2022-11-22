### 核心概念

Vue2 是响应式原理基于Object.defineProperty方法重定义对象的 getter 与 setter。vue3 则基于 Proxy 代理对象，拦截对象属性的访问与赋值过程。


Object.defineProperty存在的问题，无法拦截未定义的属性或者删除某个属性，进而需要使用$set等方式来实现响应式。

而Proxy则从根本上解决了这个问题

- Object.defineProperty

```js
const initData = { _name: 'liming' }
Object.defineProperty(initData, 'name', {
    get() {
      console.log('get')
      return initData._name
    },
    set(v) {
      console.log('set')
      initData._name = v
    }
  })

console.log(initData.name)
```

```js
const initData = { value: 1 }
const data = { }

Object.keys(initData).forEach(key => {
  Object.defineProperty(data, key, {
    get() {
      return initData[key]
    },
    set(v) {
      initData[key] = v
    }
  })
})
```

- Proxy

```js
const initData = { value: 1 }

const proxy = new Proxy(initData, {
  get(target, key) {
    return target[key]
  },
  set(target, key, value) {
    return Reflect.set(target, key, value)
  }
})
```

### 安装
```js
npm init vite-app <project-name>

cd <project-name>

npm install

npm run dev
```
### 新特性

#### 1. createApp

- main.ts

```ts
import { createApp } from 'vue'
import App from './App.vue'

createApp(App).mount('#app')
```

在2.X版本中创建一个vue 实例是通过 new Vue()来实现的,所以如果想创建多个实例实际上是不可能的。

到了3.X中则是通过使用createApp这个API返回一个应用实例。

```ts
import { createApp } from 'vue'
import App from './App.vue'
import App1 from './App1.vue'

createApp(App).mount('#app')
createApp(App1).mount('#app1')
```
以上则可以实现挂载多个应用实例上。

#### 2. defineAsyncComponent
创建一个只有在需要时才会加载的异步组件。
```js
import { defineAsyncComponent } from 'vue'

const AsyncComp = defineAsyncComponent(() =>
  import('./components/AsyncComponent.vue')
)

app.component('async-component', AsyncComp)
```

**好处：**

（1）不会将组件和其他文件打包在一起
（2）使用该方法来异步加载所需页面来提升加载速度

**使用场景**

适合于项目比较大的情景
#### 3. Teleport

将组件的 DOM 结构“传送”到指定的节点，脱离组件的父子关系，例如弹框的渲染。

```js
<div class="app">
  <teleport to="body">
    <input />
  </teleport>
</div>
```

#### 4. 多根节点
```js
<template>
  <header>...</header>
  <main v-bind="$attrs">...</main>
  <footer>...</footer>
</template>
```
不再需要像vue2必须一个根节点

#### 5. v-if 和 v-for 优先级
在同一元素上使用的 v-if 和 v-for 优先级已更改

即 if 比 for 的优先级更高了

#### 6. Composition API(组合式API)

Composition API是vue3 最有意义的一部分内容。

我们先来认识这几个新的 api（蓝色为重点内容，必须掌握）：

##### 6.1 setup 函数
（1）setup函数是一个新的组件选项，作为在组件内使用，是Composition API的入口

（2）之前的data中定义的变量，生命周期以及methods都可以放置在内

（3）setup是在创建组件之前执行，初始化props,紧接着就调用setup函数，setup 内是访问不到this的。setup => beforeCreate => data => created

(4) 具有hooks随处使用的特性，也是mixins的强化版，比mixin更加灵活。

##### 6.2 reactive | readonly | shallowReactive
- **reactive用于创建响应式对象。**
```js
const info = reactive({ title: '' })
info.title = '标题'
```
这里需要注意的是 赋值行为 是否会变量丢失响应式现象：

```js
const data = reactive([])
// 模拟接口获取数据
setTimeout(() => {
  data = [{ name: 'liming' }, { name: 'lilei' }]
})
```
如果像上面一样赋值，则data会失去响应式，如果再模板中使用该数组，则data并随着变化。

解决方式：

```js
data.push(...list) // 这样就不会丢失响应式
```
```js
const obj = reactive({ data: [] })
obj.data = list
```
所以要避免直接赋值，丢失响应式的情况。

- **readonly**

接受一个对象,只读模式，深度只读模式。
```js
const data = readonly({ value: 0 })
data.value++ // 则不会增加，因为只读模式
```

- **shallowReactive 浅度响应式**

```js
const data = shallowReactive({ count: 0, obj: { value: 1 } })

const add = () => {
  data.obj.value++
  console.log(data.obj.value)
}
```
打印data.obj.value是逐渐递增的，但是由于是shallowReactive定义，所以只是浅度响应式，所以data.obj.value如果在模板上显示是不会递增的。

- **shallowReadonly 浅度只读**

```js
const data = shallowReadonly({ count: 0, obj: { value: 1 } })

    onMounted(() => {
      data.count++
      data.obj.value++
    })
```

##### 6.3 toRaw | markRaw
##### 6.4 ref | toRef | unref | toRefs | isRef | shallowRef | triggerRef
  
- **ref 常用于包装基本类型值为响应式对象**
```js
const box = ref(0)
console.log(box.value) // 0
```
但是在模板中使用box的话，直接使用box即可，不必使用.value

- **toRef 为特定的property创建ref，则应当使用 toRef**

```js
import { toRef } from 'vue'
setup(props) {
  const title = toRef(props, 'title')
  console.log(title.value)
}
```
所以即使 props中没有该属性，toRef 也会返回一个可用的 ref。这使得它在使用可选 prop 时特别有用

- **toRefs 为原响应式对象包含的property生成ref，则可以用来解构**

```js
const data = reactive({
  name: 'liming',
  age: '18'
})
const { name } = toRefs(data)

console.log(name.value) // liming
```

- **isRef 检查值是否为一个 ref 对象**

```js
const refName = toRef(data, 'sex')
console.log(isRef(refName))
```

- **unref 如果参数是一个 ref，则返回内部值，否则返回参数本身。**
```js
val = isRef(val) ? val.value : val
```

- **shallowRef 创建一个跟踪自身.value变化的ref，但是并非响应式**
```js
const obj = ref({})
console.log(obj.value) // Proxy {} 响应式对象
const obj = shallowRef({})
console.log(obj.value) // {} 非响应式对象
```

- **triggerRef 手动执行与 shallowRef 关联的任何作用(effect)**

```js
const obj = shallowRef({ name: 'lihua' })

watchEffect(() => {
  console.log(obj.value.name) // 第一次打印 lihua
})

obj.value.name = 'timer' // 不会再次执行，因为是浅度响应式
```

但是如果加上 triggerRef

```js
const obj = shallowRef({ name: 'lihua' })

watchEffect(() => {
  console.log(obj.value.name) // 第一次打印 lihua
})

// 打印两次
// 第一次 lihua
// 第二次 liming

obj.value.name = 'liming'
triggerRef(obj)
```


##### 6.5 effect | watch | watchEffect
对数据源的更新订阅，一旦订阅的数据发生变化，将自动执行副作用函数
###### effect
```js
effect(() => {
  console.log('count.value', count.value)
})
```
###### watch

**1. 如果第一个参数是函数，则以返回值的变化为副作用执行的条件**
```js
watch(
  () => count.value > 2,
  () => console.log('我的数值已经大于2')
)
// 则第一个函数的返回值是个boolean,所以当布尔值变化时才会执行
```

**2. 如果第一个参数是一个函数，则副作用的参数就是该返回值**

```js
watch(
  () => count.value,
  value => console.log(value)
  // 则这里的value就是第一个函数的返回值
)
```

当然也可以
```js
watch(
  () => count.value,
  (value, prev) => console.log(value, prev)
)
```

**3.第一个参数如果是一个值，必须是一个响应式的对象，或者是具有getter方法的值**

- 错误写法：

```js
watch(
  count.value, // 会报错
  value => {
    console.log(value)
  }
)
```

- 正确写法

```js
watch(
  count, // 正确写法
  value => {
    console.log(value)
  }
)
```
所以说 如果第一个参数是一个响应式的对象，则以对象的属性变化为副作用执行的条件。

###### watchEffect

watchEffect 立即执行传入的一个函数，它不像watch一样，必须第一个参数作为条件。函数同时响应式追踪其依赖，并在其依赖变更时重新运行该函数。
```js
watchEffect(() => console.log('watchEffect', count.value))
```
需要注意的是，effect,watchEffect 一定是基于某一个响应式对象的**属性**有变化才会执行watchEffect副作用函数。例如一定是 监听count.value有变化才会执行，如果是count则不会执行watchEffect,因为count是一个ref对象，它并未发生变化。


##### 6.6 computed
基于响应式数据生成衍生数据，且衍生数据会同步响应式数据的变化
```js
const computedCount = computed(() => count.value + 2)
```

##### 6.7 基于 setup 方法使用的生命周期钩子同样有对应更新

![](https://files.mdnice.com/user/12101/9b26c134-7801-44fc-bb33-94ab51712e69.png)



### script setup

顶层的绑定会被暴露给模板,包括import进来的方法。

```js
<template>
  <p>我是todoList</p>
  <div>{{count}}</div>
  <button @click="add">增加</button>
</template>

<script setup lang="ts">
  import { ref } from 'vue'

  const count = ref(0)
  const add = () => {
    count.value++
  }
</script>
```

### Composition Api 更好的拆分组件

```js
<template setup>
  <img alt="Vue logo" src="./assets/logo.png" />
  <div>{{count}}</div>
  <div>{{doubleCount}}</div>
  <button @click="add">增加</button>
</template>
```

以上在vue2.x版本中，变量count,doubleCount、方法add等必须定义在当前文件中。依赖this上下文。（当然mixin可以实现，但是mixin有它自己的缺点）。

而现在ref,computed等都是在vue中单独引入

- useTodo.ts

```js
import { ref, computed } from 'vue'
export function useTodos () {
  let count: any = ref(1)

  const doubleCount = computed(() => {
    return count.value * 2
  })

  const add = () => {
    count.value += 1
  }
  return { count, doubleCount, add }
}
```

```js
<template setup>
  <img alt="Vue logo" src="./assets/logo.png" />
  <div>{{count}}</div>
  <div>{{doubleCount}}</div>
  <button @click="add">增加</button>
</template>

<script setup lang="ts">
import { useTodos } from './components/useTodo'
const { count, doubleCount, add } = useTodos()
</script>
```
因为 ref 和 computed 等功能都可以从 Vue 中全局引入，所以我们就可以把组件进行任意颗粒度的拆分和组合，这样就大大提高了代码的可维护性和复用性。

### style中的新特性

在style可以通过v-bind函数，直接在css中使用js变量。

```js
let count: any = ref(1)
let color: any = ref('red')
const add = () => {
  count.value += 1
  color.value = count.value > 3 ? 'blue' : 'red' 
}

<style scope>
  .count {
    color: v-bind(color)
  }
</style>
```

### 响应式原理


![原理的对比](https://files.mdnice.com/user/12101/30f44ada-d921-4567-b8c4-f9a5a9a16b8b.png)

### 定制响应式数据

```js
<template setup>
  <img alt="Vue logo" src="./assets/logo.png" />
  <button @click="add">增加</button>
  <div v-for="(item, index) in todos" :key="index">
    <p>{{item.name}}---{{item.count}}</p>
  </div>
</template>

<script setup lang="ts">
import { addFun } from './components/add'
const { todos, add } = addFun()
</script>
```
- add.ts
```js
import { ref } from 'vue'
export function addFun() {
  let todos = ref([{ name: 'liming', count: -1 }])
  const add = () => {
    todos.value.push({
      name: 'todo' + todos.value.length,
      count: todos.value.length
    })
  }

  return { todos, add }
}
```

这里想说明的是，add.ts返回出来的todos是响应式的数据。即使在父组件中引入 add.ts后，解构出的todos也是响应式的。

这里也同样说明了可以利用这种响应式机制，去封装独立的函数。

### defineProps
在 script setup 中必须使用 defineProps 和 defineEmits API 来声明 props 和 emits.
```js
<template>
  <h1>{{ msg }}</h1>
  <div @click="clickThis">1111</div>
</template>

<script setup lang="ts">
  // 采用ts专有声明，无默认值
  defineProps<{
    msg: string,
    num?: number
  }>()
  
  // 采用ts专有声明，有默认值
  interface Props {
      msg?: string
      labels?: string[]
  }
  const props = withDefaults(defineProps<Props>(), {
      msg: 'hello',
      labels: () => ['one', 'two']
  })

  // 非ts专有声明
  defineProps({
    msg: String,
    num: {
      type:Number,
      default: ''
    }
  })
</script>
```

### defineEmits
子组件向父组件传递事件并传参
```js
<template>
  <div @click="clickThis">点我</div>
</template>

<script setup lang="ts">
  /*ts专有*/
  const emit= defineEmits<{
    (e: 'click', num: number): void
  }>()
  /*非ts专有*/
  const emit= defineEmits(['click'])
  
  const clickThis = () => {
    emit('click',2)
  }
</script>
```

### defineExpose

传统的写法，我们可以在父组件中，通过 ref 实例的方式去访问子组件的内容，但在 script setup 中，该方法就不能用了，setup 相当于是一个闭包，除了内部的 template 模板，谁都不能访问内部的数据和方法。
如果需要对外暴露 setup 中的数据和方法，需要使用 defineExpose
- 子组件
```js
<script setup lang="ts">
import { ref } from 'vue'
const myCount = ref(1)

defineExpose({
  myCount
})
</script>
```

-父组件

```js
<template setup>
  <sub-info :info="info" @onClickBtn="changeCount" ref="subInfoRef"></sub-info>
</template>

<script setup lang="ts">
import { ref } from 'vue';
const subInfoRef = ref(null)

const changeCount = (info: any) => {
  console.log('info', info)
  const sub: any = subInfoRef.value
  console.log('subInfo', sub.myCount)
}
</script>

```

### vue3.0函数式编程优点

- vue3.0写法
![vue3.0](https://files.mdnice.com/user/12101/81dd1967-b8a2-4a80-80b3-2f42966fe049.png)

-vue2.0写法
![vue2.0](https://files.mdnice.com/user/12101/5e6e6de0-9968-4374-9a29-f92d3ccc46e8.png)

vue2是将mounted，data，computed，watch之类的方法作为一个对象的属性进行导出。

vue3新增了一个名为setup的入口函数，computed, watch, onMounted等方法都需要从外部import

函数式编程为组件的编写提供了更高的灵活度与可读性，并且更符合一个前端编写者的习惯。

在vue2中，watch、computed、data、method等API都是直接作为对象的属性，传给vue实例的。这意味着，我们开发者在开发时，脑中需要给这个对象的不同属性（data、method、mounted之类的）建立联系。但一旦代码规模变大，这种联系就非常吃力了，这集中表现在大型vue组件代码的可读性很低。我想，每个维护过1000+行vue组件的开发者都会有所体会。

而在vue3中，我们可以像写一个方法一样去写这个组件的JS逻辑部分，使用import来按需引入。这样的好处显而易见，首先就是我们需要写的代码量少了，其次就是我们可以封装更多的子函数、引用更多的公共函数去维护我们的代码，第三就是代码的可读性变高了。（当然，我们的打包体积也会变小）
