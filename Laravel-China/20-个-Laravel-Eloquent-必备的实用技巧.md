# 20 个 Laravel Eloquent 必备的实用技巧

![file](https://lccdn.phphub.org/uploads/images/201804/14/1/5WVEMTARFV.png?imageView2/2/w/1240/h/0)

Eloquent ORM 看起来是一个简单的机制，但是在底层，有很多半隐藏的函数和鲜为人知的方式来实现更多功能。在这篇文章中，我将演示几个小技巧。

### 1\. 递增和递减

要代替以下实现：

```php
$article = Article::find($article_id);
$article->read_count++;
$article->save();

```

你可以这样做：

```php
$article = Article::find($article_id);
$article->increment('read_count');

```

以下这些方法也可以实现：

```php
Article::find($article_id)->increment('read_count');
Article::find($article_id)->increment('read_count', 10); // +10
Product::find($produce_id)->decrement('stock'); // -1

```

* * * * *

### 2\. 先执行 X 方法，X 方法执行不成功则执行 Y 方法

Eloquent 有相当一部分函数可以把两个方法结合在一起使用, 例如 『 请先执行 X 方法，  X 方法执行不成功则执行 Y 方法 』。

实例 1 -- `findOrFail()`:

要替代以下代码的实现：

```php
$user = User::find($id);
if (!$user) { abort (404); }

```

你可以这样写：

```php
$user = User::findOrFail($id);

```

实例 2 -- `firstOrCreate()`:

要替代以下代码的实现：

```php
$user = User::where('email', $email)->first();
if (!$user) {
  User::create([
    'email' => $email
  ]);
}

```

这样写就可以了：

```php
$user = User::firstOrCreate(['email' => $email]);

```

* * * * *

### 3\. 模型的 boot() 方法

在一个 Eloquent 模型中，有个神奇的地方，叫 `boot()`，在那里，你可以覆盖默认的行为：

```php
class User extends Model
{
    public static function boot()
    {
        parent::boot();
        static::updating(function($model)
        {
            // 写点日志啥的
            // 覆盖一些属性，类似这样 $model->something = transform($something);
        });
    }
}

```

在创建模型对象时设置某些字段的值，大概是最受欢迎的例子之一了。 一起来看看在创建模型对象时，你想要生成 [UUID 字段](https://github.com/webpatser/laravel-uuid) 该怎么做。

```php
public static function boot()
{
  parent::boot();
  self::creating(function ($model) {
    $model->uuid = (string)Uuid::generate();
  });
}

```

* * * * *

### 4\. 带条件与排序的关联关系

定义关联关系的一般方式：

```php
public function users() {
    return $this->hasMany('App\User');
}

```

你知道吗？也可以在上面的基础上增加 `where` 或者 `orderBy`?\
举个例子，如果你想关联某些类型的用户，同时使用 email 字段排序，你可以这样做：

```php
public function approvedUsers() {
    return $this->hasMany('App\User')->where('approved', 1)->orderBy('email');
}

```

* * * * *

### 5\. 模型特性：时间、追加等

Eloquent模型有些参数，使用类的属性形式。最常用是：

```php
class User extends Model {
    protected $table = 'users';
    protected $fillable = ['email', 'password']; // 可以被批量赋值字段，如 User::create() 新增时，可使用字段
    protected $dates = ['created_at', 'deleted_at']; // 需要被Carbon维护的字段名
    protected $appends = ['field1', 'field2']; // json返回时，附加的字段
}

```

不只这些，还有：

```php
protected $primaryKey = 'uuid'; // 更换主键
public $incrementing = false; // 设置 不自增长
protected $perPage = 25; // 定义分页每页显示数量（默认15）
const CREATED_AT = 'created_at';
const UPDATED_AT = 'updated_at'; //重写 时间字段名
public $timestamps = false; // 设置不需要维护时间字段

```

还有更多,我只列出了一些有意思的特性，具体参考文档 [abstract Model class](https://github.com/laravel/framework/blob/5.6/src/Illuminate/Database/Eloquent/Model.php) 了解所有特性.

* * * * *

### 6\. 通过 ID 查询多条记录

所有人都知道  `find()`  方法，对吧？

```php
$user = User::find(1);

```
我十分意外竟然很少人知道这个方法可以接受多个 ID 的数组作为参数：

```php
$users = User::find([1,2,3]);

```

* * * * *

### 7\. WhereX

有一种优雅的方式能将这种代码：

```php
$users = User::where('approved', 1)->get();

```

转换成这种：

```php
$users = User::whereApproved(1)->get();

```

对，你没有看错，使用字段名作为后缀添加到 `where` 后面，它就能通过魔术方法运行了。

另外，在 Eloquent 里也有些和时间相关的预定义方法：

```php
User::whereDate('created_at', date('Y-m-d'));
User::whereDay('created_at', date('d'));
User::whereMonth('created_at', date('m'));
User::whereYear('created_at', date('Y'));

```

* * * * *

### 8\. 通过关系排序

一个复杂一点的「技巧」。你想对论坛话题按最新发布的帖子来排序？论坛中最新更新的主题在最前面是很常见的需求，对吧？

首先，为主题的最新帖子定义一个单独的关系：

```php
public function latestPost()
{
    return $this->hasOne(\App\Post::class)->latest();
}

```

然后，在控制器中，我们可以实现这个「魔法」：

```php
$users = Topic::with('latestPost')->get()->sortByDesc('latestPost.created_at');

```

* * * * *

### 9\. Eloquent::when() -- 不再使用 if-else

很多人都喜欢使用"if-else"来写查询条件，像这样：

```php
if (request('filter_by') == 'likes') {
    $query->where('likes', '>', request('likes_amount', 0));
}
if (request('filter_by') == 'date') {
    $query->orderBy('created_at', request('ordering_rule', 'desc'));
}

```

有一种更好的方法 -- 使用 `when()`

```php
$query = Author::query();
$query->when(request('filter_by') == 'likes', function ($q) {
    return $q->where('likes', '>', request('likes_amount', 0));
});
$query->when(request('filter_by') == 'date', function ($q) {
    return $q->orderBy('created_at', request('ordering_rule', 'desc'));
});

```

它可能看上去不是很优雅，但它强大的功能是传递参数：

```php
$query = User::query();
$query->when(request('role', false), function ($q, $role) {
    return $q->where('role_id', $role);
});
$authors = $query->get();

```

* * * * *

### 10\. BelongsTo 默认模型

假如有一个 Post 模型附属于 Author 模型，在 Blade 模板里可以写作如下代码：

```php
{{ $post->author->name }}

```

但是如果作者被删除了，或者因为某些原因未设置？你就会得到一个错误信息，类似「不存在的对象属性」。

那么，你可以这么避免它：

```php
{{ $post->author->name ?? '' }}

```

但是你可以在  Eloquent 关系模型级别做到这种效果：

```php
public function author()
{
    return $this->belongsTo('App\Author')->withDefault();
}

```

这个例子中，如果帖子没有作者的话，`author()` 关系方法会返回一个空的 `App\Author` 模型对象。

此外，我们还可以为默认模型分配一个默认的属性值。

```php
public function author()
{
    return $this->belongsTo('App\Author')->withDefault([
        'name' => 'Guest Author'
    ]);
}

```

* * * * *

### 11\. 通过赋值函数进行排序

想象一下你有这样的代码:

```php
function getFullNameAttribute()
{
  return $this->attributes['first_name'] . ' ' . $this->attributes['last_name'];
}

```
现在,你想要通过 "full_name" 进行排序? 发现是没有效果的:

```php
$clients = Client::orderBy('full_name')->get(); //没有效果

```
解决办法很简单.我们需要在获取结果后对结果进行排序.

```php
$clients = Client::get()->sortBy('full_name'); // 成功!

```

注意的是方法名称是不相同的 -- 它不是orderBy,而是sortBy

* * * * *

### 12\. 全局作用域下的默认排序

如果你想要 `User::all()` 总是按照 `name` 字段来排序呢？ 你可以给它分配一个全局作用域。让我们回到 `boot()` 这个我们在上文提到过的方法：

```php
protected static function boot()
{
    parent::boot();

    // 按照 name 正序排序
    static::addGlobalScope('order', function (Builder $builder) {
        $builder->orderBy('name', 'asc');
    });
}

```

扩展阅读 [查询作用域](https://laravel.com/docs/5.6/eloquent#query-scopes) 。

* * * * *

### 13\. 原生查询方法

有时候，我们需要在 Eloquent 语句中添加原生查询。 幸运的是，确实有这样的方法。

```php
// whereRaw
$orders = DB::table('orders')
    ->whereRaw('price > IF(state = "TX", ?, 100)', [200])
    ->get();

// havingRaw
Product::groupBy('category_id')->havingRaw('COUNT(*) > 1')->get();

// orderByRaw
User::where('created_at', '>', '2016-01-01')
  ->orderByRaw('(updated_at - created_at) desc')
  ->get();

```

* * * * *

### 14\. 复制：复制一行的副本

很简单。说明不是很深入，下面是复制数据库实体（一条数据）的最佳方法：

```php
$task = Tasks::find(1);
$newTask = $task->replicate();
$newTask->save();

```

* * * * *

### 15\. Chunk() 方法之大块数据

与 Eloquent 不完全相关,它更多的关于 Collection (集合),但是对于处理大数据集合,仍然是很有用的。你可以使用 chunk() 将这些数据分割成小数据块

修改前:

```php
$users = User::all();
foreach ($users as $user) {
    // ...

```

你可以这样做:

```php
User::chunk(100, function ($users) {
    foreach ($users as $user) {
        // ...
    }
});

```

* * * * *

### 16\. 创建模型的时候增加额外操作

我们都知道这条 Artisan 命令：

```php
php artisan make:model Company

```

但是你知道有三个十分有用的标志符可用于生成模型相关文件

```php
php artisan make:model Company -mcr

```

-   -m 创建迁移文件
-   -c 创建控制器文件
-   -r 为控制器添加资源操作方法

* * * * *

### 17\. 调用 save 方法的时候指定 updated_at

你知道  `->save()` 方法可以接受参数吗? 我们可以通过传入参数阻止它的默认行为：更新  `updated_at`  的值为当前时间戳。 
```php
$product = Product::find($id);
$product->updated_at = '2019-01-01 10:00:00';
$product->save(['timestamps' => false]);

```

这样，我们成功在  `save`  时指定了  `updated_at`  的值。

* * * * *

### 18\. update() 的结果是什么？

你是否想知道这段代码实际上返回什么？

```php
$result = $products->whereNull('category_id')->update(['category_id' => 2]);

```

我是说，更新操作是在数据库中执行的，但 `$result` 会包含什么？

答案是受影响的行。 因此如果你想检查多少行受影响， 你不需要额外调用其他任何内容 -- `update()` 方法会给你返回此数字。

* * * * *

### 19\. 把括号转换成 Eloquent 查询

如果你有个 `and` 和 `or` 混合的 SQL 查询，像这样子的：

```php
... WHERE (gender = 'Male' and age >= 18) or (gender = 'Female' and age >= 65)

```

怎么用 Eloquent 来翻译它呢？ 下面是一种错误的方式：

```php
$q->where('gender', 'Male');
$q->orWhere('age', '>=', 18);
$q->where('gender', 'Female');
$q->orWhere('age', '>=', 65);

```

顺序就没对。正确的打开方式稍微复杂点，使用闭包作为子查询：

```php
$q->where(function ($query) {
    $query->where('gender', 'Male')
        ->where('age', '>=', 18);
})->orWhere(function($query) {
    $query->where('gender', 'Female')
        ->where('age', '>=', 65);
})

```

* * * * *

### 20\. 复数参数的 orWhere

终于，你可以传递阵列参数给 `orWhere()`。平常的方式：

```php
$q->where('a', 1);
$q->orWhere('b', 2);
$q->orWhere('c', 3);

```

你可以这样做：

```php
$q->where('a', 1);
$q->orWhere(['b' => 2, 'c' => 3]);

```

* * * * *

我很确定还有更多隐藏的秘诀，但我希望至少上面的其中一些对你来说是新的。