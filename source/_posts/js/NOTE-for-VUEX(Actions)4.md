---
title: NOTE-for-VUEX(Actions)4
date: 2017-09-19 12:14:28
tags:
- js
updated: 2018-07-09 16:14:28

---



### 引入

> 由于mutation均是同步的事务，当你能调用了两个包含异步回调的 mutation 来改变状态，你怎么知道什么时候回调和哪个先回调呢？

### 概念

Action 类似于 mutation，不同在于：**Action 提交的是 mutation**，而不是直接变更状态。**Action
可以包含任意异步操作**。**对commit进行打包**
<!-- more -->
### 创建Action

Action 接受一个context对象，该对象和store实例有着相同的属性和方法，但不是store本身,这也是一个回调函数

            //传入和store实例含有相同方法和属性的对象context
            increment(context){
                context.commit('increment')
                
            },
            
            //或者使用ES6中的参数解构,来简化代码
            increment({commit}){
                commit('increment')
            }

### 组件中分发Action

分发类型

    // 以载荷形式分发
    store.dispatch('incrementAsync', {
      amount: 10
    })

    // 以对象形式分发
    store.dispatch({
      type: 'incrementAsync',
      amount: 10
    })

直接在组件实例中methods中

     //直接分发
                add:function () {
                    return this.$store.dispatch('increment')
                }

使用mapActions()来分发

        import { mapActions } from 'vuex'
        export default {

    methods:{
               //直接分发
                add:function () {
                    return this.$store.dispatch('increment')
                },

    //使用mapAction分发
                ...mapActions([
     
     // 映射 this.increment() 为 this.$store.dispatch('increment')          
             
                ]),

    //或者另取名字,传入一个对象
                    ...mapActions({

    // 映射 this.add() 为 this.$store.dispatch('increment')
                       

      })

    }

    ｝

### 组合Action（[异步对象Promise](https://medium.com/sunnoy/es6ä¸­promiseå¯¹è±¡-bb850c07cfed)）

一段时间后进行触发

    actions: {
      actionA ({ commit }) {
        return new Promise((resolve, reject) => {
          setTimeout(() => {
            commit('someMutation')
            resolve()
          }, 1000)
        })
      }
    }

使用

    //发布后，会立即进行触发resolve绑定的回调函数then()的第一个参数
    store.dispatch('actionA').then(() => {
      // ...
    })

或者进行Action的叠加

    actions: {
      // ...
      actionB ({ dispatch, commit }) {
        return dispatch('actionA').then(() => {
          commit('someOtherMutation')
        })
      }
    }
