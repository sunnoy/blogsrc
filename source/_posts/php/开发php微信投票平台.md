---
title: 开发php微信投票平台
date: 2017-09-16 12:14:28
tags:
- laravel
- php
- wechat
updated: 2017-09-17 16:14:28

---

# 使用easywechat

### laravel依赖安装

    composer require "overtrue/laravel-wechat:~4.0"

安装完成后创建配置文件

    php artisan vendor:publish --provider="Overtrue\LaravelWeChat\ServiceProvider"

在config/wechat.php中配置微信公众号的一些东西

    ‘app_id’ => env(‘WECHAT_APPID’, ‘###’), 
     ‘secret’ => env(‘WECHAT_SECRET’, ‘###’), 
     ‘token’ => env(‘WECHAT_TOKEN’, ‘###’), 
     ‘aes_key’ => env(‘WECHAT_AES_KEY’, ‘####’),
<!-- more -->
配置wechat路由

    Route::any(‘/wechat’, ‘WechatController@serve’)->name(‘wechat’);

配置Csrf中间件在app/Http/Middleware/VerifyCsrfToken.php，除去wechat路由

    protected $except = [
     ‘wechat’,
     ];

创建控制器WeChatController

    php artsian make:controller WeChatController

写入下面代码

    use Log;

    class WeChatController extends Controller
    {

    /**
     * 处理微信的请求消息
     *
     * @return string
     */
     public function serve()
     {
     Log::info(‘request arrived.’); # 注意：Log 为 Laravel 组件，所以它记的日志去 Laravel 日志看，而不是 EasyWeChat 日志

    $app = app(‘wechat.official_account’);
     $app->server->push(function($message){
     return “欢迎关注 overtrue！”;
     });

    return $app->server->serve();
     }
    }

# 接入图灵机器人

使用GuzzleHttp

### 依赖安装

需要使用http请求库guzzlehttp

    composer require guzzlehttp/guzzle

在[图灵官网](http://www.tuling123.com/)上申请帐号并获取APIkey

在WechatController中注入wechat依赖

    public function serve(Application $wechat)
<!-- more -->
添加消息处理setMessageHandler方法，并嵌套switch语句

    $wechat->server->setMessageHandler(function ($message)
        case 'event':
                        switch ($message->Event) {
                            case 'subscribe':
                                return "HI !数";
                                break;
                            default:
                                // code...
                                break;
                        }
         
                   break;

### 接入图灵机器人

在setMessageHandler方法中添加回复用户文本，并将文本发到图灵机器人api接口，并接受返回的数据，进行处理后返回给用户处理内容

获取用户输入内容

    case ‘text’:
     
    $urlcon = $message->Content;

使用GuzzleHttp客户端向图灵机器人发送POST请求

    $client = new Client();
     
    $result = $client->request(‘POST’, ‘
    , [
     ‘form_params’ => [
     ‘key’ => ‘f7b6e44c70ea46f3972d95e7bd044789’,
     ‘info’ => $urlcon,
     ‘userid' => $message->FromUserName,
     ]
     ]);

获取返回的数据，在得到内容后利用json_decode()方法转化为json数据对象

    $content = $result->getBody()->getContents();
     $content = json_decode($content);
     

图灵api会返回text，url，以及list内容，下面进行处理

    $text = $content->text;
     if (!empty($content->url)) {
     $urll = $content->url;
     };
     if (!empty($content->list)) {
     $list = $content->list;
     }
     $tuling = $text;
     if (!empty($urll)) {
     $tuling = $text . $urll;
     } elseif (!empty($list)) {
              $tuling = $text . $list;
             } elseif (!empty($urll) && !empty($list)) {
                      $tuling = $text . $urll . $list;
                     }

将处理过的数据通过 new Text()方法返回给用户

    return new Text(['content' => $tuling]);
       break;

# 保存用户图片

**添加文件系统**

首先添加文件夹/storage/app/vote，在config/filesystems.php中,添加vote驱动器，

    disks’ => [
     ‘vote’ => [
     ‘driver’ => ‘local’,
     ‘root’ => storage_path(‘app/vote’),
     ],],

**在switch方法中添加**

    case ‘image’:

    ……….

    break;
<!-- more -->
**获取用户发送的图片url**

    $url = $message->PicUrl;

通过调用自定义方法saveImage()对图片进行保存

    $file = $this->saveImage($url);

方法saveImage()

    public function saveImage($url)
     {
     $ch = curl_init();
     curl_setopt($ch, CURLOPT_URL, $url);
     curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
     curl_setopt($ch, CURLOPT_CONNECTTIMEOUT, 300);
     $file = curl_exec($ch);
     curl_close($ch);

    给图片使用随机字符串命名后进行保存
     $fileSave = Storage::disk(‘vote’)->put(date(“m-d — “) . microtime() . mt_rand() . “.jpg”, $file, ‘public’);
     return $fileSave;
     }

通过时间段限制以及保存文件成功与否的返回布尔类型值进行返回用户反馈

    if (date(‘Hi’) >= 0000 && date(‘Hi’) <= 1245) {
     $file = $this->saveImage($url);
     if ($file) {
     return “Congratulations ! 截图保存成功 ! “;
     } else {
     return “Whoops ! 截图保存失败”;
     }
     } else {
    //
     return “Whoops ! 请在每天的00:00到12:45发送截图哈”;
     }

# 压缩用户图片

**zipFiles命令**

项目主目录打开终端输入命令

    php artisan make:command zipFiles

该命令会新建类app/Console/Commands/zipFiles.php

在该类中首先定义命令名称

    protected $signature = ‘zipFiles’;
<!-- more -->
在handle方法中定义命令所执行的内容，**exec()方法用来执行Linux命令**

    public function handle()
     {
     exec(“cd /var/www/we/storage/app/vote && zip -r ShuiZhiJianCeZhan.zip ./*”);
     }

**deleteFiles命令**

控制台输入

    php artisan make:command deleteFiles

在app/Console/Commands/deleteFiles.php类中添加命令名称

    protected $signature = ‘deleteFiles’;

在handle()方法中命令执行内容

    public function handle()
     {
     exec(“cd /var/www/we/storage/app/vote && rm -rf *”);
     }

**在app/Console/Kernel.php中的私有属性$commands去注册命令**

    protected $commands = [
            Commands\zipFiles::class,
         
            Commands\deleteFiles::class,
        ];

# 发送邮件

所有邮件都需要guzzlehttp依赖


创建Mailables类，下面命令会在app/Mail/下创建voteSendEmail.php类

    voteSendEmail

在该voteSendEmail.php中的build方法定义了发送邮件HTML试图，主题，抄送，附件等

    public function build()
     {
     return $this->view(‘emails.email’)
     ->cc([‘
    ’])
     ->subject(‘水质监测站投票’)
     ->attach(storage_path(‘app/vote/ShuiZhiJianCeZhan.zip’));
     }
<!-- more -->
要想在邮件视图中添加数据，需要在该类的构造方法中传递数据给类中的共有变量，下面会把文件驱动中的文件进行统计文件数量，传递给共有变量$num

    public $num;
    public function __construct()
        {
            $fileNum = Storage::disk('vote')->files();
            $this->num = count($fileNum);
        }

在视图emails.email中展示$num变量，要在邮件视图中内嵌图片，可以使用laravel自动在email
templates创建内置的$message变量中的embed方法


邮件视图

    <body>
    <div align=”center”> 
    <h1>水质监测站投票</h1> 
    <h3>投票数为: 
    <span style=”font-style: italic;color: red;font-weight:bold;font-size: xx-large”>{{ $num-1 }}</span>
    张票</h3> 

    <img src=”{{ $message->embed(@storage_path(‘app/vote.png’)) }}”>

    </div> 
    </body>

使用lluminate\Support\Facades\Mail来发送邮件

    Mail::to(‘###@#####’)->send(new voteSendEmail());

**本例通过artisan命令来发送邮件**

创建sendEmails命令

    php artisan make:command SendEmails

在app/Console/Commands/SendEmails.php中定义命令名称

    protected $signature = ‘sendEmails’;

在handle()方法中定义发送邮件命令

    public function handle()    {        
          Mail::to('###@#####')->send(new voteSendEmail());
        }

在app/Console/Kernel.php中的私有属性$commands去注册命令

    protected $commands = [
     Commands\SendEmails::class,
     ];

# 定时任务配置

对artisan命令进行定时任务

首先需要在服务器上安装cron


然后使用命令创建定时认为列表


打开后填入下来laravel命令，该命令每分钟执行一次

<!-- more -->
在laravel项目中打开app/Console/Kernel.php，并在schedule方法中加入定时认为命令

    protected function schedule(Schedule $schedule)
     {
     //zip files
     $schedule->command(‘zipFiles’)
     ->timezone(‘Asia/Shanghai’)
     ->dailyAt(‘12:45’);
     //send emails
     //$schedule->command(‘sendEmails’)
     // ->timezone(‘Asia/Shanghai’)
     //->dailyAt(‘13:30’);
     //delete files
     $schedule->command(‘deleteFiles’)
     ->timezone(‘Asia/Shanghai’)
     ->dailyAt(‘23:50’);
     }


