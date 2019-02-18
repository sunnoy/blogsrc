---
title: NOTE-for-VUEX(Mutations)3
date: 2017-09-19 12:14:28
tags:
- js
updated: 2018-07-09 16:14:28

---



### 引入

> *更改 Vuex 的 store 中的状态的唯一方法是提交 mutation。Vuex 中的 mutations 非常类似于事件：每个 mutation
> 都有一个字符串的 事件类型 (type) 和 一个 回调函数 (handler)。这个回调函数就是我们实际进行状态更改的地方，并且它会接受 state
作为第一个参数*
<!-- more -->
### 创建mutation在store实例中

    mutations:{
     //mutation含有increment字符串的事件类型和一个回调函数，并且把state作为他的第一个参数
     increment(state){
     //也可以传入第二个参数，对象类型的载荷payload:{amount:2}
     //increment(state,payload){
     state.count++
     //state.count += payload.amount 
     }
     }

### 在组件中提交

mutation在组件中的methods方法内填入，**mutilation提交均为同步提交**

可以直接提交


    methods:{
     add:function () {
     return this.$store.commit(‘increment’)
     }
     }

也可以使用mapMutations()函数提交


    //引入mapMutations
     import { mapMutations } from 'vuex'
        export default {

    methods:{
                ...mapMutations([
    // 映射 this.increment() 为 this.$store.commit('increment')
                    'increment'
                ])

    }

    ｝
