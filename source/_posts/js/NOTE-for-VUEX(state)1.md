---
title: NOTE-for-VUEX(state)1
date: 2017-09-19 12:14:28
tags:
- js
updated: 2018-07-09 16:14:28

---



### **laravel-mix对ES6中 “…”展开运算符支持**

首先安装babel插件


然后在项目根目录也就是.env的同级目录创建文件”.babelrc”

    //文件 .babelrc内容
    {
     “plugins”: [“transform-object-rest-spread”]
    }

**创建 VUEX 实例 store.js**

    import Vue from ‘vue’
    import vuex from ‘vuex’
    Vue.use(vuex);
    const store = new vuex.Store({
     state: {
     count: 6
     },
     mutations: {
     increment (state) {
     state.count++
     }
     },

    });
    export default store
<!-- more -->
**在根实例中引入创建的store，这样store会注入到所有的子组件中去**

    import store from ‘./store’

    const inapp = new Vue({
     el: ‘#inapp’,
     
     store
    });

**在子组件中以计算属性的方式获取store中的state**

分为直接获取和利用mapState()来获取state

    //使用计算属性来获得store中的state的数据
     computed: 
     
     //直接获取stat数据
     count () {
     return this.$store.state.count
     }
     
     
     //利用mapState来获取state
     mapState({
     //常规获取方法
     count: state => state.count,
     
     //于子组件中的数据进行运算
     countPlusLocalState (state) {
     return state.count + this.localCount
     }
     })
     
     //利用映射，计算属性的名称与 state 的子节点名称相同时，传递一个数组
     computed: mapState([
     // 映射 this.count 为 store.state.count
     ‘count’
     ])
