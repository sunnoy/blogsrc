---
title: NOTE-for-VUEX(Getters)2
date: 2017-09-19 12:14:28
tags:
- js
updated: 2018-07-09 16:14:28

---



### **使用场景**

> 有时候我们需要从 store 中的 state 中派生出一些状态

    //没有使用Getters时就要这样做
    computed: {
     doneTodosCount () {
     return this.$store.state.todos.filter(todo => todo.done).length
     }
    }
<!-- more -->
### store 的计算属性，对state中数据进行派生

> 就像计算属性一样，getters的返回值会根据它的依赖被缓存起来，且只有当它的依赖值发生了改变才会被重新计算。

    state: {
            count: 68,
            todos:[
                {id:1,content:'ete',done:true},
                {id:2,content:'watch tv',done:false},
            ]
        },

    //如此使用过后getters会暴露出getters对象
     //即store.getters.doneTodos
     getters:{
     //Getters 接受 state 作为其第一个参数,Getters 也可以接受其他 getters 作为第二个参数
     //doneTodosCount: (state, getters) 

     doneTodos: state => {
                    return state.todos.filter(todo => todo.done)
                }
     }

### 组件中使用

直接在组件计算属性中使用和利用辅助函数mapGetters()

    //直接使用
    {{ doneTodosCount[0].id }}

    computed:{
                doneTodosCount () {
                    return this.$store.getters.doneTodos
                }

    }

    //利用mapGetters()
    import { mapGetters } from 'vuex'

    computed: {
                // 使用对象展开运算符将 getters 混入 computed 对象中
                ...mapGetters([
                    'doneTodos'
                ]),
                ...mapState([
                    'count'
                ])
            }


    //给getter 属性另取一个名字，使用对象形式
    mapGetters({
      // 映射 this.doneCount 为 store.getters.doneTodosCount
      doneCount: 'doneTodosCount'
    })
