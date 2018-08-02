![file](https://lccdn.phphub.org/uploads/images/201804/21/1/RqiF7U4Wud.png?imageView2/2/w/1240/h/0)

在我们的 Laravel 应用中，有几十甚至几百个视图文件再正常不过了。 在这些文件中我们需要各种用到路由地址的地方，回想一下你已经像下面这样写过多少次了:

```php
<a href="{{ route('users.show', $user) }}">{{ $user->name }}</a>

```

当我们需要对某个路由进行修改的时候，比如改成使用路由别名，或者添加一个参数，你很快就会发现你不得不使用一种高风险的操作：批量替换。

那有没有办法更科学的来解决这个痛点呢？这里介绍两种办法。

只针对 Eloquent 模型的简单改进方式
-------------

```php
<?php

namespace App;

class User {

  protected $appends = [
    'url'
  ];

  public function getUrlAttribute()
  {
    return route('users.show', $this);
  }
}

```

然后你的视图文件就可以这样写了：

```php
<a href="{{ $user->url }}">{{ $user->name }}</a>

```

怎么样，是不是非常的简洁明了？如果你是一个高级开发者，你也许会对下面一种用法更感兴趣。

Eloquent 模型与 URL Presenter 搭配使用
---------------------------

乍一看，与上面的用法基本差不多，只是属性访问器返回值变成了一个 Presenter 对象。

```php
<?php

namespace App;

use App\Presenters\User\UrlPresenter;

class User {

  protected $appends = [
    'url'
  ];

  public function getUrlAttribute()
  {
    return new UrlPresenter($this);
  }
}

```

```php
<?php

namespace App\Presenters\User;

use App\User;

class UrlPresenter {

    protected $user;

    public function __construct(User $user)
    {
        $this->user = $user;
    }

    public function __get($key)
    {
        if(method_exists($this, $key))
        {
            return $this->$key();
        }

        return $this->$key;
    }

    public function delete()
    {
        return route('users.delete', $this->user);
    }

    public function edit()
    {
        return route('users.edit', $this->user);
    }

    public function show()
    {
        return route('users.show', $this->user);
    }

    public function update()
    {
        return route('users.update', $this->user);
    }
}

```

然后你就可以这样在视图中使用了：

```php
<a href="{{ $user->url->show }}">{{ $user->name }}</a>

```

无论你使用以上哪种方式，视图都不必关心 URL 的生成方式，只是简单的输出模型的 URL。它的美妙之处在于，当你需要修改 URL 的时候，你仅仅只需要修改两个文件，而不是原来的几十上百个文件。