# nvm安装

  地址：https://github.com/nvm-sh/nvm
  1. curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
  2. export NVM_DIR="${XDG_CONFIG_HOME/:-$HOME/.}nvm"
     [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
  3. nvm --version
  4. nvm ls-remote

# 安装浏览器调试工具

  下载地址:https://github.com/vuejs/vue-devtools#vue-devtools
  步骤:
      1. npm install
      2. npm run build
      3. localhost:8100

# 安装脚手架
  cnpm install -g @vue/cli
  cnpm install -g @vue/cli-service-global

  vue init webpack project-name

# 创建工程
  vue create my-project
  # OR
  vue ui


# 语法

+ v-bind (简写 -> :) 常用于静态属性

缩写
  > <!-- 完整语法 -->
    <a v-bind:href="url">...</a>
    <!-- 缩写 -->
    <a :href="url">...</a>


+ v-on (简写 -> @) 用于绑定事件

缩写
  > <!-- 完整语法 -->
    <a v-on:click="doSomething()">...</a>
    <!-- 缩写 -->
    <a @click="doSomething()">...</a>


+ v-html  解析html标签
+ v-text  内容
+ v-model 数据双向绑定，一般用于表单

+ 条件渲染
  v-if
  v-else
  v-else-if
  v-show

+ 列表渲染
  v-for  设置键值 key
  ```
    <ul>
      <li v-for="(todo, index) in todos" :key="index">
        {{ todo.text }}
      </li>
    </ul>
  ```

+ 计算属性
computed -- (数据联动)

+ 监听器
watch -- (异步场景)

+ 全局组件
```
 // 全局组件
  Vue.component('item-todo', {
    props: ['listName', 'index'],
    template: '<li @click="handleLi">{{ listName }}</li>',
    methods: {
      // 触发模板方法
      handleLi: function () {
        // 自定义事件,可以将值传递过来
        this.$emit('show-value', this.index);
      },
    }
  });

  <!-- 这意味着当你使用 DOM 中的模板时，camelCase (驼峰命名法) 的 prop 名需要使用其等价的 kebab-case (短横线分隔命名) 命名：，监听show-value事件 -->
  <item-todo v-for="(item, index) in items" :key="index" :list-name="item" :index="index" @show-value="showItem">

  methods: {
    showItem: function (index) {
      alert(this.items[index]);
      this.items.splice(index, 1);
    }
  }
```

+ 局部组件
```
  var todoItem = {
    props: ['todo'],
    template: '<li>{{ todo.text }}</li>'
  })

  // 其中props的属性设置
  props: {
    title: String,
    likes: Number,
    isPublished: Boolean,
    commentIds: Array,
    author: Object,
    callback: Function,
    contactsPromise: Promise // or any other constructor
  }

  var app = new Vue({
    // 注册组件
    components: {
      'todo-item': todoItem,
      'component-b': ComponentB
    }
  })
```

# vue-router 路由管理
```
  // 路由链接
  <router-link to="/info">Info</router-link>


  import Home from "./views/Home.vue";

  routes: [
      // 定义组件
      {
        path: "/",
        name: "home",
        component: Home
      },
      {
        path: "/about",  // 路径
        name: "about",   // 别名
        component: () =>
          import("./views/About.vue")
      }
    ]
```

# 服务端交互-axios

+ 安装

1)：npm install axios --save
2)：npm install qs.js --save　　//它的作用是能把json格式的直接转成data所需的格式

```
1) main.js

import Vue from 'vue'
import axios from 'axios'
import qs from 'qs'

Vue.prototype.$axios = axios    //全局注册，使用方法为:this.$axios
Vue.prototype.qs = qs           //全局注册，使用方法为:this.qs

2) 组件中
created(){
    this.$axios({
        method:'post',
        url:'api',
        data:this.qs.stringify({    //这里是发送给后台的数据
              userId:this.userId,
              token:this.token,
        })
    }).then((response) =>{          //这里使用了ES6的语法
        console.log(response)       //请求成功返回的数据
    }).catch((error) =>{
        console.log(error)       //请求失败返回的数据
    })
}

```


# vuex 状态管理
```
  // 驱动应用的数据源
  state: {
    count: 0,

  },

  mutations: {
    // 定义
    increment(state) {
      state.count++;
    }
  },
  // 响应在 view 上的用户输入导致的状态变化。
  actions: {

  }
```
模块中import store from "@/store.ts";
使用   `soore`
      `store.commit("increment")`
      `store.state.count`