![Laravel best practices](https://github.com/alexeymezenin/laravel-best-practices/raw/master/images/logo-english.png?raw=true)


这篇文章并不是什么由 Laravel 改编的 SOLID 原则、模式等。

只是为了让你注意你在现实生活的 Laravel 项目中最常忽略的内容。

### 单一责任原则

一个类和一个方法应该只有一个职责。

错误的做法：

```php
public function getFullNameAttribute()
{
    if (auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified()) {
        return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
    } else {
        return $this->first_name[0] . '. ' . $this->last_name;
    }
}
```

正确的做法：

```php
public function getFullNameAttribute()
{
    return $this->isVerifiedClient() ? $this->getFullNameLong() : $this->getFullNameShort();
}

public function isVerifiedClient()
{
    return auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified();
}

public function getFullNameLong()
{
    return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
}

public function getFullNameShort()
{
    return $this->first_name[0] . '. ' . $this->last_name;
}
```

### 强大的模型 & 简单控制器

如果你使用查询构造器或原始 SQL 来查询，请将所有与数据库相关的逻辑放入 Eloquent 模型或存储库类中。

坏：

```php
public function index()
{
    $clients = Client::verified()
        ->with(['orders' => function ($q) {
            $q->where('created_at', '>', Carbon::today()->subWeek());
        }])
        ->get();

    return view('index', ['clients' => $clients]);
}
```

好：

```php
public function index()
{
    return view('index', ['clients' => $this->client->getWithNewOrders()]);
}

class Client extends Model
{
    public function getWithNewOrders()
    {
        return $this->verified()
            ->with(['orders' => function ($q) {
                $q->where('created_at', '>', Carbon::today()->subWeek());
            }])
            ->get();
    }
}
```

### 验证

将验证从控制器移动到请求类。

很常见但不推荐的做法：

```php
public function store(Request $request)
{
    $request->validate([
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
        'publish_at' => 'nullable|date',
    ]);

    ....
}
```

最好是这样：

```php
public function store(PostRequest $request)
{    
    ....
}

class PostRequest extends Request
{
    public function rules()
    {
        return [
            'title' => 'required|unique:posts|max:255',
            'body' => 'required',
            'publish_at' => 'nullable|date',
        ];
    }
}
```

###业务逻辑应该在服务类中

一个控制器必须只有一个职责，因此应该将业务逻辑从控制器移到服务类。

坏：

```php
public function store(Request $request)
{
    if ($request->hasFile('image')) {
        $request->file('image')->move(public_path('images') . 'temp');
    }
    
    ....
}
```

好：

```php
public function store(Request $request)
{
    $this->articleService->handleUploadedImage($request->file('image'));

    ....
}

class ArticleService
{
    public function handleUploadedImage($image)
    {
        if (!is_null($image)) {
            $image->move(public_path('images') . 'temp');
        }
    }
}
```

### 不要重复你自己（DRY）

尽可能重用代码。 SRP（单一职责原则）正在帮助你避免重复。当然，这也包括了 Blade 模板、Eloquent 的范围等。

坏：

```php
public function getActive()
{
    return $this->where('verified', 1)->whereNotNull('deleted_at')->get();
}

public function getArticles()
{
    return $this->whereHas('user', function ($q) {
            $q->where('verified', 1)->whereNotNull('deleted_at');
        })->get();
}
```

好：

```php
public function scopeActive($q)
{
    return $q->where('verified', 1)->whereNotNull('deleted_at');
}

public function getActive()
{
    return $this->active()->get();
}

public function getArticles()
{
    return $this->whereHas('user', function ($q) {
            $q->active();
        })->get();
}
```

###最好倾向于使用 Eloquent 而不是 Query Builder 和原生的 SQL 查询。要优先于数组的集合

Eloquent 可以编写可读和可维护的代码。此外，Eloquent 也拥有很棒的内置工具，比如软删除、事件、范围等。

比如你这样写：

```sql
SELECT *
FROM `articles`
WHERE EXISTS (SELECT *
              FROM `users`
              WHERE `articles`.`user_id` = `users`.`id`
              AND EXISTS (SELECT *
                          FROM `profiles`
                          WHERE `profiles`.`user_id` = `users`.`id`) 
              AND `users`.`deleted_at` IS NULL)
AND `verified` = '1'
AND `active` = '1'
ORDER BY `created_at` DESC
```

还不如这样写：

```php
Article::has('user.profile')->verified()->latest()->get();
```

### 批量赋值

比如你这样写：

```php
$article = new Article;
$article->title = $request->title;
$article->content = $request->content;
$article->verified = $request->verified;
// Add category to article
$article->category_id = $category->id;
$article->save();
```

Good:

```php
$category->article()->create($request->all());
```

[🔝 Back to contents](###contents)

### 不要在 Blade 模板中执行查询并使用关联加载（N + 1 问题）

不好的地方在于，这对于100 个用户来说，等于执行 101 个 DB 查询：

```php
@foreach (User::all() as $user)
    {{ $user->profile->name }}
@endforeach
```

下面的做法，对于 100 个用户来说，仅仅只执行 2 个 DB 查询：

```php
$users = User::with('profile')->get();

...

@foreach ($users as $user)
    {{ $user->profile->name }}
@endforeach
```

### 与其花尽心思给你的代码写注释，还不如对方法或变量写一个描述性的名称

坏：

```php
if (count((array) $builder->getQuery()->joins) > 0)
```

尚可：

```php
// Determine if there are any joins.
if (count((array) $builder->getQuery()->joins) > 0)
```

好：

```php
if ($this->hasJoins())
```

### 不要把 JS 和 CSS 放在 Blade 模板中，也不要将任何 HTML 放在 PHP 类中

坏：

```php
let article = `{{ json_encode($article) }}`;
```

尚可：

```php
<input id="article" type="hidden" value="{{ json_encode($article) }}">

Or

<button class="js-fav-article" data-article="{{ json_encode($article) }}">{{ $article->name }}<button>
```

最好的方法是使用在 Javascript 中这样来传输数据：

```php
let article = $('###article').val();
```

The best way is to use specialized PHP to JS package to transfer the data.

### 在代码中使用配置和语言文件、常量，而不是写死它

坏：

```php
public function isNormal()
{
    return $article->type === 'normal';
}

return back()->with('message', 'Your article has been added!');
```

好：

```php
public function isNormal()
{
    return $article->type === Article::TYPE_NORMAL;
}

return back()->with('message', __('app.article_added'));
```

### 使用社区接受的标准的 Laravel 工具

最好使用内置的 Laravel 功能和社区软件包，而不是其他第三方软件包和工具。因为将来与你的应用程序一起工作的开发人员都需要学习新的工具。另外，使用第三方软件包或工具的话，如果遇到困难，从 Laravel 社区获得帮助的机会会大大降低。不要让你的客户为此付出代价！

任务 | 标准工具 | 第三方工具
------------ | ------------- | -------------
授权 | Policies | Entrust, Sentinel and other packages
前端编译 | Laravel Mix | Grunt, Gulp, 3rd party packages
开发环境 | Homestead | Docker
部署 Laravel Forge | Deployer and other solutions
单元测试 | PHPUnit, Mockery | Phpspec
浏览器测试 | Laravel Dusk | Codeception
数据库操作 | Eloquent | SQL, Doctrine
模板 | Blade | Twig
数据操作 | Laravel collections | Arrays
表单验证 | Request classes | 3rd party packages, validation in controller
认证 | Built-in | 3rd party packages, your own solution
API 认证 | Laravel Passport | 3rd party JWT and OAuth packages
创建 API | Built-in | Dingo API and similar packages
数据库结构操作 | Migrations | Working with DB structure directly
局部化 | Built-in | 3rd party packages
实时用户接口 | Laravel Echo, Pusher | 3rd party packages and working with WebSockets directly
Generating testing data | Seeder classes, Model Factories, Faker | Creating testing data manually
生成测试数据 | Laravel Task Scheduler | Scripts and 3rd party packages
数据库 | MySQL, PostgreSQL, SQLite, SQL Server | MongoDB

### 遵循Laravel命名约定

遵循 PSR 标准。 另外，请遵循 Laravel 社区接受的命名约定：

类型 | 规则 | 正确示例 | 错误示例
------------ | ------------- | ------------- | -------------
Controller | singular | ArticleController | ~~ArticlesController~~
Route | plural | articles/1 | ~~article/1~~
Named route | snake_case with dot notation | users.show_active | ~~users.show-active, show-active-users~~
Model | singular | User | ~~Users~~
hasOne or belongsTo relationship | singular | articleComment | ~~articleComments, article_comment~~
All other relationships | plural | articleComments | ~~articleComment, article_comments~~
Table | plural | article_comments | ~~article_comment, articleComments~~
Pivot table | singular model names in alphabetical order | article_user | ~~user_article, articles_users~~
Table column | snake_case without model name | meta_title | ~~MetaTitle; article_meta_title~~
Model property | snake_case | $model->created_at | ~~$model->createdAt~~
Foreign key | singular model name with _id suffix | article_id | ~~ArticleId, id_article, articles_id~~
Primary key | - | id | ~~custom_id~~
Migration | - | 2017_01_01_000000_create_articles_table | ~~2017_01_01_000000_articles~~
Method | camelCase | getAll | ~~get_all~~
Method in resource controller | [table](https://laravel.com/docs/master/controllers###resource-controllers) | store | ~~saveArticle~~
Method in test class | camelCase | testGuestCannotSeeArticle | ~~test_guest_cannot_see_article~~
Variable | camelCase | $articlesWithAuthor | ~~$articles_with_author~~
Collection | descriptive, plural | $activeUsers = User::active()->get() | ~~$active, $data~~
Object | descriptive, singular | $activeUser = User::active()->first() | ~~$users, $obj~~
Config and language files index | snake_case | articles_enabled | ~~ArticlesEnabled; articles-enabled~~
View | snake_case | show_filtered.blade.php | ~~showFiltered.blade.php, show-filtered.blade.php~~
Config | snake_case | google_calendar.php | ~~googleCalendar.php, google-calendar.php~~
Contract (interface) | adjective or noun | Authenticatable | ~~AuthenticationInterface, IAuthentication~~
Trait | adjective | Notifiable | ~~NotificationTrait~~

### 尽可能使用更短、更易读的语法

坏：

```php
$request->session()->get('cart');
$request->input('name');
```

好：

```php
session('cart');
$request->name;
```

更多示例：

通用语法 | 更短、更可读的语法
------------ | -------------
`Session::get('cart')` | `session('cart')`
`$request->session()->get('cart')` | `session('cart')`
`Session::put('cart', $data)` | `session(['cart' => $data])`
`$request->input('name'), Request::get('name')` | `$request->name, request('name')`
`return Redirect::back()` | `return back()`
`is_null($object->relation) ? $object->relation->id : null }` | `optional($object->relation)->id`
`return view('index')->with('title', $title)->with('client', $client)` | `return view('index', compact('title', 'client'))`
`$request->has('value') ? $request->value : 'default';` | `$request->get('value', 'default')`
`Carbon::now(), Carbon::today()` | `now(), today()`
`App::make('Class')` | `app('Class')`
`->where('column', '=', 1)` | `->where('column', 1)`
`->orderBy('created_at', 'desc')` | `->latest()`
`->orderBy('age', 'desc')` | `->latest('age')`
`->orderBy('created_at', 'asc')` | `->oldest()`
`->select('id', 'name')->get()` | `->get(['id', 'name'])`
`->first()->name` | `->value('name')`

### 使用 IoC 容器或 facades 代替新的 Class

新的 Class 语法创建类时，不仅使得类与类之间紧密耦合，还加重了测试的复杂度。推荐改用 IoC 容器或 facades。

坏：

```php
$user = new User;
$user->create($request->all());
```

好：

```php
public function __construct(User $user)
{
    $this->user = $user;
}

....

$this->user->create($request->all());
```

### 不要直接从 .env 文件获取数据

将数据传递给配置文件，然后使用辅助函数 config() 在应用程序中使用数据。

坏：

```php
$apiKey = env('API_KEY');
```

Good:

```php
// config/api.php
'key' => env('API_KEY'),

// Use the data
$apiKey = config('api.key');
```

### 以标准格式存储日期，必要时就使用访问器和修改器来修改日期格式

坏：

```php
{{ Carbon::createFromFormat('Y-d-m H-i', $object->ordered_at)->toDateString() }}
{{ Carbon::createFromFormat('Y-d-m H-i', $object->ordered_at)->format('m-d') }}
```

坏：

```php
// Model
protected $dates = ['ordered_at', 'created_at', 'updated_at']
public function getSomeDateAttribute($date)
{
    return $date->format('m-d');
}

// View
{{ $object->ordered_at->toDateString() }}
{{ $object->ordered_at->some_date }}
```

### 其他良好做法

千万不要在路由文件中放置任何逻辑。

在 Blade 模板中最小化 vanilla PHP 的使用。
本文章首发在 Laravel China 社区

Stay Hungry, Stay Foolish.
