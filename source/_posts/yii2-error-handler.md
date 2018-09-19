---
title: Yii2倒腾系列：自定义错误处理
tags:
  - Yii2
  - PHP
categories:
  - Yii2
date: '2018-04-24 16:00'
abbrlink: 40357
---

# 起因

Yii2框架，在dev环境下时，错误提示是很友好的，但是，如果我们的代码在线上或者在预发布环境上，有时我们不需要将错误显示的那么明显，当然这时我们可以设置环境为prod，这样就不会有错误提示了，但是一旦出了错误，对于开发人员的排查也就比较耗时耗力了，这时，如果程序出错了，便能给开发人员发送一个邮件，告诉开发人员用户的请求地址，请求参数，错误文件，错误的行数，这样也就可以让开发人员在第一时间解决bug，而展示给用户的仅仅是一条我们定义的错误提示，这样是不是会更好呢？

<!--more-->

## errorHandler组件设置

Yii2为我们提供了errorHandler组件，我们只需要设置一下当出现错误的时候，应该路由到的控制器即可，在main.php或main-local.php中的components中添加

~~~
        'errorHandler' => [
            'errorAction' => 'site/error',
        ],
~~~

其次，开启errorHandler 这个组件，需要，关闭debug，同时环境切换的dev下，即在index.php中，将*YII_DEBUG*和*YII_ENV* 定义如下

~~~
defined('YII_DEBUG') or define('YII_DEBUG', false);
defined('YII_ENV') or define('YII_ENV', 'dev');
~~~

最后，假如我们将错误处理交给SiteController下actionError() （site/error）这个方法处理，请保证SiteController或者SiteController的父类控制器里面没有下面的内容

~~~
    public function actions()
    {
        return [
            'error' => [
                'class' => 'yii\web\ErrorAction',
            ],
        ];
    }
~~~

上面的 *actions* 方法会将*actionError()*这个方法重新指向 *yii\web\ErrorAction* 。

# 将错误信息用邮件发给开发人员

Yii2框架本身的错误提示页面比较友好，所以我便将页面内容拿过来，并做了一点修改。

1. 首先新建一个error对应view模板页面（views/site/error.php），将Yii2本身的错误异常处理页面模板exception.php拷贝过来并修改，相信各位 这点简单的操作 肯定不是什么问题的。

2. 将actionError()修改如下

   ~~~
   	public function actionError()
   	{
           http_response_code(200);
           $exception = Yii::$app->errorHandler->exception;
           $statusCode = Yii::$app->response->statusCode;
           // 过滤404
           // 此时在 file_put_contents的时候，由于jquery.map.js不存在而报404的错误
           if ($statusCode != 404) {
               $exception_key = Yii::$app->params['ExceptionMailList'];
               $request_url = $_SERVER['REQUEST_SCHEME'] . "://" . $_SERVER['HTTP_HOST'] . $_SERVER['REQUEST_URI'];
               $params = json_encode(Yii::$app->request->post());
               $content = $this->renderPartial('exception', ['exception' => $exception,
                   'handler' => Yii::$app->errorHandler,
                   'request_url' => $request_url,
                   'params'    =>  $params
               ]);
               foreach (Yii::$app->params['ExceptionNotifyList'] as $email) {
                   Email::begin()->setType('exception')
                       ->setEmail($email)
                       ->setUsername('lixiaoyu')
                       ->setValue('content', $content)
                       ->send();
               }
           }

           return  FunctionHelper::errorJson("An internal server error occurred");
   	}
   ~~~

   上面代码仅仅是渲染异常页面，然后获取渲染后的页面，并作为邮件内容发送给设置的开发人员的邮箱，这样，便能在第一时间收到异常信息并处理了。