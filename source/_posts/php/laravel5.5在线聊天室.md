---
title: laravel5.5在线聊天室
date: 2017-09-10 12:14:28
tags:
- laravel
- php
- 聊天室
updated: 2017-09-09 19:14:28

---

# laravel5.5在线聊天室

## 路由和数据库

![聊天室](https://qiniu.li-rui.top/聊天室.jpg)

### 定义数据库以及迁移


`文件web.php 和 model.php`

创建模型以及数据库迁移
```bash
    php artisan make:model Message -m
```
<!-- more -->

编写数据库迁移在function on方法

```php
    Schema::create(‘messages’, function (Blueprint $table) {
     $table->increments(‘id’);
     $table->text(‘message’);

    //编写外键
     $table->integer(‘user_id’)->unsigned();
     $table->timestamps();
     });

```

建立数据库关系，对model Message

    class Message extends Model
    {
     protected $fillable= [‘message’];
     public function user(){

    //定义一对多数据库关系,正向关系
     return $this->belongsTo(User::class);
     }
    }

对model User

    //增加$fillable属性，以便可以进行批量填写数据
    protected $fillable = [
     ‘name’, ‘email’, ‘password’,
     ];

    public function messages(){
            //定义一对多数据库关系，反向关系
            return $this->hasMany(Message::class);
        }

生成数据库迁移


    //如要回滚迁移

    //回滚所有迁移，然后再进行迁移’

> **定义路由**

在routes/web.php中定义

    Route::get(‘/messages’, function () {

    //返回全部数据表中数据库结合
     return App\Message::with(‘user’)->get();
    })->middleware(‘auth’);

    Route::post('/messages', function () {
        //获取当前认证用户
        $user = Auth::user();

    //调用User model 中messages()方法然后链式调用create()方法，create()方法会增加一条数据然后返回该数据的实例对象，该方法需要所增加的字段在模型中定义$fillable属性
        $message = $user->messages()->create([
            'message' => request()->get('message')
        ]);
        //触发事件进行广播
        broadcast(new MessagePosted($message,$user))->toOthers();
        return ['status' => 'ok'];
    })->middleware('auth');


## 事件和广播

利用broadcasting

> **依赖安装**

    pusher依赖安装：composer require pusher/pusher-php-server “~3.0”

在[pusher](https://pusher.com/)中注册获取AppKeys


在.env配置文件中配置

    BROADCAST_DRIVER=pusher

    PUSHER_APP_ID=
    PUSHER_APP_KEY=
    PUSHER_APP_SECRET=
<!-- more -->
还需要在config/broadcasting.php配置数组connections中，添加

    ‘pusher’ => 
    [ 
    ‘driver’ => ‘pusher’, 
    ‘key’ => env(‘PUSHER_APP_KEY’), 
    ‘secret’ => env(‘PUSHER_APP_SECRET’), 
    ‘app_id’ => env(‘PUSHER_APP_ID’), 
    ‘options’ => 
        [ 
           //添加集群配置
            ‘cluster’ => “ap1” 
        ], 
    ]

    以及在config/app.php中取消对

    App\Providers\BroadcastServiceProvider::class,的注释

<!-- more -->

> **创建事件**

    php artisan make:event MessagePosted

为了让创建的事件可以被广播出去需要添加ShouldBroadcast接口

    class MessagePosted 
    public $message;

    //属性
     可以指定 非默认的队列名称  

    Message $message
    $this->message = $message;



    return new PresenceChannel('chatroom');


> **配置路由**

在routes/channels.php中定义广播路由路由

    //第一个参数是广播频道名称，后面是一个回调函数，函数内给出验证条件然后返回布尔类型的值
    //所有的回调函数中第一个参数均是当前所认证的用户，也可以诸如model依赖

    Broadcast::channel(‘chatroom’, function ($user)
     { 
          return $user;
     });

    //特别针对Presence Channels的授权路由，其回调函数不可以返回true在用户通过授权的情况下，应该返回包含用户信息的数组，因为js应用要求使用用户信息，如果对于没有通过授权的用户应该返回false或者null
    //Broadcast::channel('chat.{roomId}', function ($user, $roomId) {
    //    if ($user->canJoinRoom($roomId)) {

    //通过认证九返回用户数据 ，没有通过就没有返回，即返回null       
    //return ['id' => $user->id, 'name' => $user->name];
    //    }
    //});

> **广播事件**

广播事件使用event函数

    event(new MessagePosted($message,$user));

    //该路由会接受一个POST请求，然后把请求中的数据保存至数据库，并且将请求中的数据和用户通过事件广播到所有的客户端
    Route::post('/messages', function () {
        
        //store the new message
        $user = Auth::user();
        $message = $user->messages()->create([
            'message' => request()->get('message')
        ]);
        
    //Announce that a new message has been posted
    //event(new MessagePosted($message,$user)); 
    // 这样会给所打开某一网页的用户发送广播，在这个聊天室应用中一个用户就会收到两条一样的消息，所以使用  
    broadcast(new MessagePosted($message,$user))->toOthers();
       
    })->middleware('auth');

> **接收广播**

首先安装前端依赖


并在resources/assets/js/bootstrap.js中引入依赖

    import Echo from ‘laravel-echo’

    window.Pusher = require(‘pusher-js’);

    window.Echo = new Echo({
     
     broadcaster: ‘pusher’,
     key: ‘###’,
     cluster: ‘###’

    });

接着就可以使用接收接收广播的方法了

在resources/assets/js/app.js中const app = new Vue({});方法内加入

    created(){

    //特别针对Presence channels这里使用的是join方法，即检索一个频道的实例，对于其他频道使用

    Echo.join(‘chatroom’)

    //here方法将返回一个当前所有查看某个网页用户的数组    
    .here((users) => {
          this.usersInRoom = users;
        })

    //joining方法将返回新加入某一网页用户的对像
        .joining((user) =>{
         this.usersInRoom.push(user)
        })

    //leaving方法将返回新离开某一网页的用户对象
        .leaving((user) => {
       //过滤数组方法filter，除去当前传入的用户对象，返回剩下的用户对象
        this.usersInRoom = this.usersInRoom.filter(u => u !==user)
       })

    //listen方法为接收广播的方法，方法的第一个参数为时间广播频道的名称，默认为事件的类名，第二个参数为接受广播中的数据，该数据就是事件类中所包含的共有属性，

       .listen(‘MessagePosted’,(e) => {
        this.messages.push({
          message:e.message.message,
          user:e.user
           });
        });
     }

> **js应用之间事件**

不经过服务器，比如对方正在输入。。。

发起者：

    Echo.channel(‘chat’)
     .whisper(‘typing’, {
     对方正在输入。。。
     });

接收者

    Echo.channel(‘chat’)
     .listenForWhisper(‘typing’, (e) => {
     console.log(e);//对方正在输入。。。
     });

## 视图

balde and vue.js and element.js

> **前端依赖安装**

安装默认前端资源以及element.js


    //element安装

    resources/assets/js/bootstrap.js或者
    //resources/assets/js/app.js

    import ElementUI from 'element-ui'
    import 'element-ui/lib/theme-default/index.css'

    //VUe调用插件
    Vue.use(ElementUI);


> **vue组件创建**

**chat-composer组件**

在resources/views/中创建chat.blade.php模板

    (‘layouts.app’)

    (‘content’)
     <div class=”container”>
     <div id=”app”>
     <div class=”container”>

    <h1>聊天室</h1>

    实时显示聊天室内人数，利用Presence channels
     <h3 class=”badge”>当前聊天室内有@{{ usersInRoom.length }}个人</h3>

    显示聊天记录vue组件
     <chat-log :messages=”messages”></chat-log>

    //接收输入信息，绑定一个未来会被激发的messagesent事件
     <chat-composer v-on:messagesent=”addMessage”></chat-composer>
     </div>
     </div>
     </div>
<!-- more -->

创建ChatComposer.vue，在resources/assets/js/components/中

该组件包含一个输入框和一个按钮，输入框监听key.enter时间，按钮监听click事件，并均对应组件中sendMessage方法，该方法会调用vue中$emit触发事件的方法，触发balde模板中定义的messagesent事件所对应的app.js中addMessage方法，并把组件中v-model绑定的输入框输入内容通过$emit方法传递出去。

    <template>
    <div class=”chat-composer”>
     <input type=”text” @keyup.enter=”
    ” placeholder=”输入信息” v-model=”messageText”>
     <button class=”btn btn-primary” @click=”
    ”>发送</button>
    </div>
    </template>
    <script>
     export default {
     data(){
     return {

    //定义一个变量用来绑定input输入内容
     messageText:’’
     }
     },
     methods:{
     
    (){

    //触发messagesent事件，并把input控件内容和当前用户名传递出去
     this.$emit(‘messagesent’,{
     message:this.messageText,
     user:{
     name:$(‘.navbar-right .dropdown-toggle’).text()
     }
     });

    //清空输入框内容
     this.messageText=’’
     }
     }
     }
    </script>

    <style lang=”css”>
     .chat-composer {
     display: flex;
     }
     .chat-composer input{
     flex-shrink: 1;
     flex-grow: 1;
     flex-basis: auto;
     }
     .chat-composer button {
     border-radius: 3%;
     }
    </style>

在app.js中建立vue实例

其中addMessage方法把接受一个messagesent事件传递过来的数据参数，并在浏览器中更新实例中messages数组返回到视图中，然后调用Ajax方法axios.post()请求写入数据库。

    el: '#app',
        data:{

    //全局定义数组messages
            messages: [],
            usersInRoom:[]
        },

    methods:{
     addMessage(parvl){
     //add to existing messages
     this.messages.push(parvl);
     //to data bsase
     axios.post(‘/messages’, parvl).then(response => {
     // Do whatever;
     })

    },

**chat-log组件**

该组件定义一个props:[‘messages’]，并且将全局定义的messages数组放入其中。组件中嵌入了一个chat-message组件，并调用v-for方法进行迭代出messages数组，将迭代出的单个数据元素传递给chat-message中porps的message变量。

    <template>
     <div class=”chat-log”>
     <! — v-bind(:) 组件自身变量=”所绑定的父组件数据” --->
     <chat-message v-for=”message in messages” :message=”message”></chat-message>

    <div class=”empty” v-show=”messages.length === 0">
     这里什么都没有哇

    </div>
     </div>

    </template>

    <script>
     export default {
     props:[‘messages’],
     }
    </script>

    <style lang=”css”>
     .chat-log .chat-message:nth-child(even){
     background-color: #cccccc;
     }
     .empty{
     padding: 1rem;
     text-align: center;
     }
    </style>

**chat-message组件**

该组件定义message的props来接收messages中的迭代数据，并把把它显示出来，分别显示出消息内容message.message，发消息人message.user.name。

    <template>

    <div class=”chat-message”>
     <p>{{ message.message }}</p>
     <small>{{ message.user.name }}</small>
     </div>

    </template>
    <script>
     export default {
     props:[‘message’]
     }
    </script>

    <style>
    </style>

> **app.js中vue实例**

导入创建的vue组件，实例初始化就实现created()方法和Larvel
Echo加入频道方法。其中created()方法调用axios.get()方法向数据库请求全部聊天数据，并把获取的数据填入messages数组

    //导入ElementUI插件
    import ElementUI from ‘element-ui’
    import ‘element-ui/lib/theme-default/index.css’

    require(‘./bootstrap’);

    window.Vue = require(‘vue’);

    //引入ElementUI插件
    Vue.use(ElementUI);
    /**
     * Next, we will create a fresh Vue application instance and attach it to
     * the page. Then, you may begin adding components to this application
     * or customize the JavaScript scaffolding to fit your unique needs.
     */

    Vue.component(‘my-vuetable’, require(‘./components/Example.vue’));
    Vue.component(‘chat-message’, require(‘./components/ChatComponent.vue’));
    Vue.component(‘chat-log’, require(‘./components/ChatLog.vue’));
    Vue.component(‘chat-composer’, require(‘./components/ChatComposer.vue’));

    const app = new Vue({
     el: ‘#app’,
     data:{
     messages: [],
     usersInRoom:[]
     },
     methods:{
     addMessage(parvl){
     //add to existing messages
     this.messages.push(parvl);
     //to data bsase
     axios.post(‘/messages’, parvl).then(response => {
     // Do whatever;
     })

    },

    },

    //向数据库发送请求，获取全部聊天数据
    created(){
     axios.get(‘/messages’).then(response => {
     this.messages = response.data;
     });

    Echo.join(‘chatroom’)
     .here((users) => {
     this.usersInRoom = users;
     })
     .joining((user) =>{
     this.usersInRoom.push(user)
     })
     .leaving((user) => {
     //过滤数组，除去当前传入的对象
     this.usersInRoom = this.usersInRoom.filter(u => u !==user)
     })
     .listen(‘MessagePosted’,(e) => {
     this.messages.push({
     message:e.message.message,
     user:e.user
     });
     });
     }
    });


