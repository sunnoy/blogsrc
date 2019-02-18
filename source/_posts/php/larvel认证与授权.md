---
title: laravel5.5事件与监听
date: 2017-09-19 12:14:28
tags:
- laravel
- php
updated: 2018-07-09 16:14:28

---


### About gates

add **isAdmin()** function in model User.php

    public function isAdmin(){
            return false;
        }

defined in the** App\Providers\AuthServiceProvider **class using the* ***Gate
facade**

    use App\User;

    Gate::define('admin', function (User $user, Post $post){

    return $user->
    ;

    //$user->id == $post->user_id;

    });
<!-- more -->
To authorize an action using gates, to use the **allows or denies** methods. it
just return true or false ,and

> **Note that you are not required to pass the currently authenticated user to
> these methods.**

    use Illuminate\Support\Facades\Gate;

    dd(Gate::allows('admin'));//return false

    dd(Gate::denies('admin'));//return true

    //Gate::allows('update-post', Post $post)

if in controller that use the Trait AuthorizesRequests and that has authorize
methods so we can use the authorize()

    $this->authorize('admin');//if authorize fail the method return AccessDeniedHttpException

    $this->authorize('admin', Post $post);

### About Policies

**Generating Policies**

> Policies are classes that organize authorization logic around a particular model
> or resource

    php artisan make:policy PostPolicy

    //you can generate policies use --model=model,to create the basic "CRUD" policy methods

**Registering Policies**

in app/Providers/AuthServiceProvider.php ,it has protected var

    protected $policies = [
            'App\Model' => 'App\Policies\ModelPolicy',
            
            
        ];

**Writing Policies**

**when Methods With Models**


**when Methods Without Models**


**when other code authorizing application administrators**

    public function before($user, $ability)
    {
        if ($user->isSuperAdmin()) {
            return true;
        }
    }

**Authorizing Actions Using Policies**

Via The User Model


    if ($user->can('update', $post)) {
        //
    }



Via Middleware


    use App\Post;

    Route::put('/post/{post}', function (Post $post) {
        // The current user may update the post...
    })->middleware('can:update,post');



Via Controller Helpers

    public function update(Request $request, Post $post)
        {
            $this->authorize('update', $post);

    // The current user can update the blog post...
        }


Via Blade Templates

    @can('update', $post)
        <!-- The Current User Can Update The Post -->
    @elsecan('create', $post)
        <!-- The Current User Can Create New Post -->
    @endcan

    @cannot('update', $post)
        <!-- The Current User Can't Update The Post -->
    @elsecannot('create', $post)
        <!-- The Current User Can't Create New Post -->
    @endcannot

