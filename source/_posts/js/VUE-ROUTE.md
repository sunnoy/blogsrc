---
title: VUE-ROUTE
date: 2017-09-19 12:14:28
tags:
- js
updated: 2018-07-09 16:14:28

---



### 安装

    npm install vue-router

    //开始调用
    import VueRouter from ‘vue-router’

    Vue.use(VueRouter)
<!-- more -->
### 创建router.js

    import Vue from 'vue'

    import VueRouter from 'vue-router'

    Vue.use(VueRouter);

    //
    引入定义路由组件
    import addItem from './components/addItem.vue'
    import about from './components/about.vue'
    import recent from './components/recent.vue'

    //
    定义路由,数组，每一个路由就是一个对象在数组里面
    const routes = [
        {path: '/additem', component: addItem},
        {path: '/recent', component: recent},
        {path: '/about', component: about}
    ];

    //
    创建 router 实例，然后传 `routes` 配置
    const router = new VueRouter({
        // （缩写）相当于 routes: routes
        routes
    });

    export default router;

### 根实例调用

    const inapp = new Vue({

    store,
        router
    }).$mount('#inapp');

### **视图中调用**

    //可以在组件中使用
    <p>
              <router-link to="/additem">additem</router-link>
              <router-link to="/recent">recent</router-link>
              <router-link to="/about">about</router-link>
    </p>
              <router-view></router-view>

### js中调用路由

    // 字符串
    router.push('home')

    // 对象
    router.push({ path: 'home' })

    // 命名的路由
    router.push({ name: 'user', params: { userId: 123 }})

    // 带查询参数，变成 /register?plan=private
    router.push({ path: 'register', query: { plan: 'private' }})

    具体到连接中

    //按钮视图
    <mu-bottom-nav 
    >

    <mu-bottom-nav-item value="dform" title="提交数据" icon="add"/>
    <mu-bottom-nav-item value="recents" title="提交记录" icon="description"/>

    <mu-bottom-nav-item value="about" title="关于" icon="note"/>

    </mu-bottom-nav>

    //js处理

    methods: {
                handleChange(val) {
                    this.bottomNav = val;
                    switch (val) {
                        case "dform": {
                            this.Ntitle = "提交数据";
                            this.$router.push('additem');
                            break;
                        }
                        case "recents": {
                            this.Ntitle = "提交记录";
                            this.$router.push('recent');

    break;
                        }

    case "about": {
                            this.Ntitle = "关于";
                            this.$router.push('about');

    break;
                        }

    }
                }
            }

### 命名路由

    const router = new VueRouter({
      routes: [
        {
          path: '/user/:userId',
          name: 'user',
          component: User
        }
      ]
    })

### 重定向路由，主页到某一页面

    const router = new VueRouter({
      routes: [
     //重定向根目录到/additem
        {path: '/', redirect: '/additem'}
      ]
    })
