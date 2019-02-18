---
title: larvel5.5数据库关系
date: 2017-09-19 12:14:28
tags:
- laravel
- php
updated: 2018-07-09 16:14:28

---

# larvel5.5数据库关系

## 一对一

创建profile表以及模型

    php artisan make:model Profile -m

修改数据库迁移

    Schema::create(‘profiles’, function (Blueprint $table) {
     $table->increments(‘id’);
     $table->integer(‘user_id’);
     $table->string(‘city’,64);
     $table->text(‘about’);
     $table->timestamps();
     });
<!-- more -->
生成迁移

    php artisan migrate

在app/Http/Controller/Auth/RegisterController中create方法中添加如下代码，其中User::create方法会在创建一条记录后返回这条记录的对象。

    protected function create(array $data)
     {
     $user = User::create([
     ‘name’ => $data[‘name’],
     ‘email’ => $data[‘email’],
     ‘password’ => bcrypt($data[‘password’]),
     ]);



    return $user;
     }

在resources/views/home.blade.php中添加如下代码，进行填写

    <div class=”panel-body”>
     
     (session(‘status’))
     <div class=”alert alert-success”>
     {{ session(‘status’) }}
     </div>
     


    You are logged in!
     </div>

一对一反向

在model Profile中添加如下方法

    public function user(){

     }

由profile方法user

## 一对多

创建articles迁移

    php artisan make:model Articles -m

编写迁移

    Schema::create(‘articles’, function (Blueprint $table) {
     $table->increments(‘id’);
     $table->integer(‘user_id’);
     $table->string(‘title’);
     $table->text(‘body’);
     $table->timestamps();
     });
<!-- more -->
生成迁移

    php artisan migrate

对User模型

    public function articles (){
     return $this->hasMany(Articles::class);
     }

对Articles模型

    public function user(){
     return $this->belongsTo(User::class);
     }

关于保存

    一对多，一个用户对应多篇文章
    //使用用户保存数据库
     

    //使用文章保存数据库

在home视图中展示文章

    (Auth::user()->articles as $article)

    title {{ $article->title }}<br/>
     body {{ $article->body }}<br/>



## 多对多

创建jores表

    php artisan make:model Jore -m

编写迁移

    Schema::create('jores', function (Blueprint $table) {
                $table->increments('id');
                $table->string('jore');
                $table->timestamps();
            });

创建中间表article_website迁移

    php artisan make:migration creat_user_jore_table --create=user_jore

编写迁移

    Schema::create('user_jore', function (Blueprint $table) {
                $table->increments('id');
                $table->integer('user_id');
                $table->integer('jore_id');
                $table->timestamps();
            });
<!-- more -->
生成迁移

    php artisan migrate

多对多正向

User模型，第二个参数为中间表名称

    public function jores (){
     return $this->belongsToMany(Jore::class,’user_jore’);
     }

Jore模型

    public function users(){
     return $this->belongsToMany(User::class,’user_jore’);
     }

保存多对多关系模型

    $user = \App\User::find(1);
     $user->jores()->attach($joreId);

调用多对多模型，动态方法获得数组，进行foreach循环

    {{ Auth::user()->jores[0]->jore }}
