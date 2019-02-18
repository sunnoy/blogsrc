---
title: laravel5.5事件与监听
date: 2017-09-19 12:14:28
tags:
- laravel
- php
updated: 2018-07-09 16:14:28

---


**如何理解事件与监听**

    public function a(){
     
     echo ‘yes’;
     //event
     $this->b()->c();
     }
     
     //Listeners
     public function b(){
     return $this;
     }
     public function c(){
     return $this;
     //dd
     }

<!-- more -->

**在app/Providers/EventServiceProvider.php中protected $listen[]数组中注册事件和监听**

    protected $listen = [
     ‘App\Events\UserRegistered’ => [
     ‘App\Listeners\assignRole’,
     ‘App\Listeners\sendActivationCode’,
     ],
     ];

**在app/Events/UserRegistered.php定义事件**

    public $user;
     //其构造方法用来获取事件中的传送数据，公共属性都会传递出去
     public function __construct(User $user)
     {
     $this->user = $user;
     }

**在app/Listeners/assignRole.php和sendActivationCode.php中的handle方法中定义监听**，该方法接受一个所监听的事件实例，$event就是类UserRegistered.php的实例

    public function handle(UserRegistered $event)
     {
     Log::info(‘role’,[‘user’ => $event->user]);
     }

**触发事件**，使用event()方法，该方法传入一个变量数据

    event(new UserRegistered($user));
