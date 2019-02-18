---
title: ES6中Promise对象
date: 2017-09-19 12:14:28
tags:
- js
updated: 2018-07-09 16:14:28

---



### 概念

> Promise对象代表一个异步操作，有三种状态：pending（进行中）、fulfilled（已成功）和rejected（已失败），从`pending`变为`fulfilled`和从`pending`变为`rejected`。只要这两种情况发生，状态就凝固了，不会再变了，会一直保持这个结果，这时就称为
> resolved（已定型）
<!-- more -->
Promise构造函数接受一个函数作为参数，该函数的两个参数分别是resolve和reject，这是两个函数。

**resolve**用来把promise对象的状态从“未完成”变为“成功”（即从 pending 变为 resolved

**reject**用来把promise对象的状态从“未完成”变为“失败”（即从 pending 变为 rejected）

    var promise = new Promise(function(resolve, reject) {
      // ... some code

    if (/* 异步操作成功 */){
        resolve(value);
      } else {
        reject(error);
      }
    });

### **实例与then函数**

    function timeout(ms) {
            //创建promise对象
            return new Promise((resolve, reject) => {
                setTimeout(reject, ms);
            });
        }

       //then()函数接受两个参数，第一个参数为resolve绑定的回调函数，
       // 一旦resolve函数执行就会触发then函数的第一个参数
       //第二个参数为reject绑定的回调函数，一旦reject函数执行就会触发第二个参数函数
        timeout(3000)
            
            .then(() => {
             //setTimeout(resolve, ms);触发
            alert('ok');
        },() =>{
            //setTimeout(reject, ms);触发
            alert('erro')
        });




