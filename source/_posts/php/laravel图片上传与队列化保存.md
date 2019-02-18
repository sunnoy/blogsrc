---
title: laravel图片上传与队列化保存
date: 2017-09-19 12:14:28
tags:
- laravel
- php
updated: 2018-07-09 16:14:28

---




**依赖安装**

Intervention Image图片处理库

    composer require intervention/image

    在config/app.php中添加
    Intervention\Image\ImageServiceProvider::class,
    'Image' => Intervention\Image\Facades\Image::class,

    执行命令
<!-- more -->
**文件系统设置**

在config/filesystems.php中进行如下设置

    ‘public’ => [
     ‘driver’ => ‘local’,
     //’root’ => storage_path(‘app/public’),
     ‘root’ => public_path(),
     //’url’ => env(‘APP_URL’).’/storage’,
     ‘url’ => env(‘APP_URL’),
     ‘visibility’ => ‘public’,
     ],

文件上传blade模板，在resources/views.file/index.balde.php

    <h2>Up loade file</h2>
    <form action=”{{ route(‘file.store’) }}” method=”post” enctype=”multipart/form-data”>
     {{ csrf_field() }}
     <label for=”file”>choose file</label>
     <input type=”file” name=”file” id=”file”><br/>
     <button type=”submit”>upload</button>
    </form>

路由设置在routes/web.php

    Route::post(‘/file’, ‘FileController@store’)->name(‘file.store’);
    Route::get(‘/file’, ‘FileController@index’);

控制器创建

    php artisan make:controller FileController

控制器代码

    public function index(){
     return view(‘file.index’);
     }

    public function store(Request $request){
     $file = $request->file(‘file’)->store(‘uploads’,’public’);

    //一下两行代码将用于队列任务执行
    $image = Image::make(public_path($file));

    $image->insert(public_path(‘uploads/laravel.png’))->save();

    }

**创建队列job**，该命令会创建app/Jobs/DoHeavyStuff.php

    php artisan make:job DoHeavyStuff

类DoHeavyStuff.php中含有一个构造函数，用来让队列实例接收一个参数，传递触发队列的数据；一个handle函数，在队列中用来执行任务，也就是上面那两行代码

    private $file;

    public function __construct($file)
     {
     
     $this->file = $file;
     }

    public function handle()
        {
            $image = Image::make(public_path($this->file));

    $image->insert(public_path('uploads/laravel.png'))->save();

    }

**队列驱动配置**

在.env中配置队列驱动为数据库，**更改后需要重启Web服务器**

    QUEUE_DRIVER=database

创建job表以及失败记录表

    php artisan queue:table
    php artisan queue:failed-table
    php artisan migrate

使用队列，以及命令参数

在控制器中调用队列

    DoHeavyStuff::dispatch($file)
     ->delay(Carbon::now()->addMinutes(1));

队列命令参数

    php artisan queue:work

    最多尝试几次对每一个队列
    php artisan queue:work --tries=3

    //重新启动队列

    //列出失败队列
    php artisan queue:failed
    //清空失败队列
