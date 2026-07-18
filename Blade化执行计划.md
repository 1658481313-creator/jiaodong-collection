# 胶东私人收藏 — Laravel Blade 化执行计划

## 一、Laravel 项目目录结构

```
胶东收藏/
├── app/
│   ├── Http/Controllers/
│   │   ├── Front                    # 前台控制器
│   │   │   ├── HomeController.php
│   │   │   ├── CollectionController.php
│   │   │   ├── CollectorController.php
│   │   │   ├── StoryController.php
│   │   │   ├── UpdateController.php
│   │   │   ├── ContactController.php
│   │   │   └── HelpController.php
│   │   ├── Admin/                   # 后台 API 控制器
│   │   │   ├── CollectionController.php
│   │   │   ├── CollectorController.php
│   │   │   ├── StoryController.php
│   │   │   ├── UpdateController.php
│   │   │   ├── PageController.php
│   │   │   └── DashboardController.php
│   │   └── Middleware/
│   │       └── AdminAuth.php
│   ├── Models/
│   │   ├── Collection.php
│   │   ├── Collector.php
│   │   ├── Story.php
│   │   ├── Update.php
│   │   ├── Category.php
│   │   ├── Contact.php
│   │   └── Page.php
│   └── Services/
│       ├── ViewCountService.php      # 阅读量统计（Cache::increment）
│       └── ContactService.php       # 联系表单处理
├── database/
│   ├── migrations/
│   │   ├── 001_create_categories_table.php
│   │   ├── 002_create_collectors_table.php
│   │   ├── 003_create_collections_table.php
│   │   ├── 004_create_stories_table.php
│   │   ├── 005_create_updates_table.php
│   │   ├── 006_create_contacts_table.php
│   │   ├── 007_create_pages_table.php
│   │   └── 008_create_view_counts_table.php
│   └── seeders/
│       ├── CategorySeeder.php
│       ├── CollectorSeeder.php
│       ├── CollectionSeeder.php
│       ├── StorySeeder.php
│       ├── UpdateSeeder.php
│       ├── PageSeeder.php
│       └── ViewCountSeeder.php
├── resources/
│   └── views/
│       ├── front/                     # 前台模板
│       │   ├── layouts/
│       │   │   └── app.blade.php     # 主布局
│       │   ├── partials/             # 公共组件
│       │   │   ├── header.blade.php
│       │   │   ├── mobile-menu.blade.php
│       │   │   ├── footer.blade.php
│       │   │   ├── breadcrumb.blade.php
│       │   │   ├── pagination.blade.php
│       │   │   ├── back-to-top.blade.php
│       │   │   └── stat-band.blade.php
│       │   ├── home.blade.php
│       │   ├── collection/
│       │   │   ├── index.blade.php       # 藏品列表
│       │   │   └── detail.blade.php      # 藏品详情
│       │   ├── collector/
│       │   │   ├── index.blade.php       # 收藏家列表
│       │   │   └── detail.blade.php      # 收藏家详情
│       │   ├── story/
│       │   │   ├── index.blade.php       # 故事列表
│       │   │   └── detail.blade.php      # 故事详情
│       │   ├── update/
│       │   │   ├── index.blade.php       # 动态列表
│       │   │   └── detail.blade.php      # 动态详情
│       │   ├── contact.blade.php
│       │   └── help.blade.php            # info.html 的合并版
│       └── admin/                     # 后台 Ant Design Pro（独立部署）
│           └── (Ant Design Pro SPA，不在此仓库)
├── public/
│   ├── css/
│   │   └── theme.css               # 原 pages/theme.css 直接迁移
│   ├── images/
│   │   ├── logo.png                # 原 assets/logo.png
│   │   ├── banner/                 # 原 banner-*.jpg
│   │   ├── collection/            # 原 collection-*.jpg
│   │   ├── news/                   # 原 news-*.jpg
│   │   ├── story/                  # 原 story-*.jpg
│   │   └── collector/              # 原 collector-*.jpg（如有）
│   └── storage/                    # php artisan storage:link 指向这里
│       └── app/public/uploads/     # 后台上传的图片
└── routes/
    ├── web.php                     # 前台路由
    ├── api.php                     # 后台 API 路由（给 Ant Design Pro 用）
    └── admin.php                   # 后台管理页面路由（可选）
```

---

## 二、模板拆分方案

### 2.1 主布局 `layouts/app.blade.php`

从当前各页面中提取重复部分，构成主布局：

```
+------------------------------------------+
|  <!DOCTYPE html> <html> <head>             |
|    - meta charset, viewport                |
|    - meta description (每页不同)           |
|    - <title> (每页不同)                    |
|    - theme.css                             |
|    - Tailwind CDN                          |
|    - Lucide CDN                            |
|    - 内联 style (通用 responsive + scrollbar)|
+------------------------------------------+
|  @yield('header')                         |
|  @yield('mobile-menu')                    |
+------------------------------------------+
|  <main>                                   |
|    @yield('breadcrumb')   ← 可选           |
|    @yield('content')                       |
|  </main>                                   |
+------------------------------------------+
|  @yield('footer')                          |
|  @yield('back-to-top')                    |
|  <script> lucide.createIcons() </script>   |
+------------------------------------------+
|  @stack('scripts')  ← 页面级 JS            |
+------------------------------------------+
|  </body></html>                           |
+------------------------------------------+
```

### 2.2 公共组件拆分明细

| 组件 | 来源 | 说明 |
|---|---|---|
| `partials/header.blade.php` | 所有页面 header | 接收 `$activeNav` 参数高亮当前页面，Logo 统一 |
| `partials/mobile-menu.blade.php` | 所有页面 | `toggleMobileMenu()` JS 内联，和 header 独立 |
| `partials/footer.blade.php` | 所有页面 footer | 从数据库读取分类链接（不再写死），4 列结构不变 |
| `partials/breadcrumb.blade.php` | 详情页 | 接收 `$items` 数组（[{label, url}]） |
| `partials/pagination.blade.php` | collection/updates | 接收 Laravel 分页对象，输出按钮组 |
| `partials/back-to-top.blade.php` | index 等 | 固定按钮 + scroll 事件监听 |
| `partials/stat-band.blade.php` | index | 从数据库统计收藏家数/藏品数/分类数 |

### 2.3 各页面模板与对应 Blade 变量

#### home.blade.php（原 index.html）

```
@extends('front.layouts.app')
@section('title', '胶东私人收藏 - 胶东地区私人藏品信息展示平台')
@section('description', '...')
@section('header')
    @include('front.partials.header', ['activeNav' => 'home'])
@endsection
@section('content')

    @foreach($featuredCollections as $collection)
        {{-- 藏品卡片，图片: Storage::url($collection->cover_image) --}}
    @endforeach

    @include('front.partials.stat-band', [
        'collectorCount' => $stats['collectors'],
        'collectionCount' => $stats['collections'],
        'categoryCount' => $stats['categories']
    ])
@endsection
```

**控制器传入变量：**
```
HomeController@index → $featuredCollections, $stats, $categories
```

#### collection/index.blade.php（原 collection.html）

**关键变化：** 去掉全部前端 JS 筛选/分页/搜索逻辑，改为 URL 参数驱动。

```
@extends('front.layouts.app')
@section('title', '藏品展示 - 胶东私人收藏')

@section('content')
    {{-- 第一行：标题 + 搜索/排序 --}}
    <form method="GET" action="{{ route('collections.index') }}" class="...">
        <input name="keyword" value="{{ request('keyword', '') }}">
        <select name="sort">
            <option value="latest" @selected(request('sort') === 'latest')>最新发布</option>
            <option value="az" @selected(request('sort') === 'az')>名称 A-Z</option>
        </select>
    </form>

    {{-- 第二行：分类筛选（URL 参数驱动，点击即跳转） --}}
    <a href="{{ route('collections.index') }}" class="{{ $currentCategory === null ? 'active' : '' }}">全部</a>
    @foreach($categories as $cat)
        <a href="{{ route('collections.index', ['category' => $cat->slug]) }}"
           class="{{ $currentCategory === $cat->slug ? 'active' : '' }}">
            {{ $cat->name }}
        </a>
    @endforeach

    {{-- 卡片网格 --}}
    @foreach($collections as $collection)
        <a href="{{ route('collections.show', $collection->slug) }}">
            <img src="{{ Storage::url($collection->cover_image) }}" alt="{{ $collection->title }}">
            <h3>{{ $collection->title }}</h3>
            <p>{{ Str::limit($collection->description, 80) }}</p>
        </a>
    @endforeach

    {{-- 分页（Laravel 自带） --}}
    {{ $collections->appends(request()->query())->links() }}
@endsection
```

**控制器：**
```
CollectionController@index
    - $query = Collection::published()
    - if request('category'): $query->whereHas('category', fn => fn->where('slug', request('category')))
    - if request('keyword'): $query->where('title', 'LIKE', '%'.request('keyword').'%')
    - if request('sort') === 'az': $query->orderBy('title')
    - else: $query->orderBy('created_at', 'desc')
    - return $query->paginate(20)->withQueryString()
```

#### collection/detail.blade.php（原 collection-detail.html）

```
@extends('front.layouts.app')
@section('title', "{$collection->title} - 胶东私人收藏")
@section('breadcrumb')
    @include('front.partials.breadcrumb', [
        'items' => [
            ['label' => '首页', 'url' => route('home')],
            ['label' => '藏品展示', 'url' => route('collections.index')],
            ['label' => $collection->title]
        ]
    ])
@endsection
@section('content')
    {{-- 藏品大图 + 元数据表 + 详情描述 --}}
    {{-- 同分类推荐（$relatedCollections） --}}
@endsection
```

#### 其他列表页（collectors, stories, updates）

结构与 collection/index.blade.php 相同模式：
- 筛选用 `<a>` 链接（URL 参数），不用 JS
- 分页用 Laravel `$paginator->links()`
- 排序用 `<form method="GET">` + `<select>`
- 卡片数据用 `@foreach`

#### 详情页（collector/detail, story/detail, update/detail）

- 面包屑导航：`@include('front.partials.breadcrumb')`
- 阅读量：`ViewCountService::increment($type, $id)`
- 侧边栏（story-detail, update-detail 的方案B布局）：用 Blade 的 `@section` 或直接写在详情模板里

#### contact.blade.php（原 contact.html）

- 表单 `method="POST" action="{{ route('contact.store') }}"`
- `@csrf` 保护
- 服务端验证（`$request->validate([...])`）
- 验证失败 `@error` 指令显示错误信息（替代当前 JS alert）
- 提交成功后 `return redirect()->back()->with('success', '提交成功')`

#### help.blade.php（原 info.html）

当前 info.html 有 5 个 tab（平台服务说明、常见问题、入驻指南、隐私政策、使用条款）。

Blade 化后改为路由驱动：

```
routes/web.php:
    Route::get('/help', [HelpController::class, 'index'])->name('help.index');           // 默认：平台服务说明
    Route::get('/help/service', [HelpController::class, 'service'])->name('help.service');
    Route::get('/help/faq', [HelpController::class, 'faq'])->name('help.faq');
    Route::get('/help/guide', [HelpController::class, 'guide'])->name('help.guide');
    Route::get('/help/privacy', [HelpController::class, 'privacy'])->name('help.privacy');
    Route::get('/help/terms', [HelpController::class, 'terms'])->name('help.terms');

HelpController:
    - 每个方法从 pages 表读取对应 slug 的内容
    - 传给同一个 Blade 模板（或各用独立模板）
    - 左侧导航高亮当前 tab
```

这样 `#service`、`#faq` 等 hash 导航变为真正的独立 URL，SEO 更好。

---

## 三、数据迁移步骤

### 阶段 1：数据库搭建（第 1 周）

**Step 1.1 创建项目**
```bash
composer create-project laravel/laravel 胶东收藏
cd 胶东收藏
php artisan storage:link
```

**Step 1.2 创建迁移文件**

categories 表：
```php
Schema::create('categories', function (Blueprint $table) {
    $table->id();
    $table->string('slug')->unique();
    $table->string('name');
    $table->unsignedInteger('sort_order')->default(0);
    $table->timestamps();
});
```

collectors 表：
```php
Schema::create('collectors', function (Blueprint $table) {
    $table->id();
    $table->string('slug')->unique();
    $table->string('name');
    $table->string('avatar')->nullable();
    $table->string('region', 50);
    $table->string('direction')->nullable();       // 收藏方向
    $table->text('bio')->nullable();
    $table->enum('status', ['pending', 'approved', 'rejected'])->default('approved');
    $table->timestamps();
});
```

collections 表：
```php
Schema::create('collections', function (Blueprint $table) {
    $table->id();
    $table->string('slug')->unique();
    $table->string('title');
    $table->text('description')->nullable();
    $table->string('era')->nullable();             // 朝代（自由文本）
    $table->foreignId('category_id')->constrained()->cascadeOnDelete();
    $table->foreignId('collector_id')->nullable()->constrained()->nullOnDelete();
    $table->string('cover_image')->nullable();
    $table->json('images')->nullable();             // 详情图片数组
    $table->json('metadata')->nullable();            // 扩展字段（尺寸、材质等）
    $table->enum('status', ['draft', 'published'])->default('published');
    $table->unsignedInteger('sort_order')->default(0);
    $table->unsignedInteger('view_count')->default(0);
    $table->timestamps();
});
```

stories 表：
```php
Schema::create('stories', function (Blueprint $table) {
    $table->id();
    $table->string('slug')->unique();
    $table->string('title');
    $table->string('summary')->nullable();
    $table->mediumText('content')->nullable();
    $table->enum('type', ['藏品故事', '藏家故事'])->default('藏品故事');
    $table->string('cover_image')->nullable();
    $table->string('author')->nullable();
    $table->enum('status', ['draft', 'published'])->default('published');
    $table->unsignedInteger('view_count')->default(0);
    $table->timestamp('published_at')->nullable();
    $table->timestamps();
});
```

updates 表：
```php
Schema::create('updates', function (Blueprint $table) {
    $table->id();
    $table->string('slug')->unique();
    $table->string('title');
    $table->string('summary')->nullable();
    $table->mediumText('content')->nullable();
    $table->enum('type', ['平台采访', '行业资讯', '展览信息', '市场动态'])->default('行业资讯');
    $table->string('cover_image')->nullable();
    $table->string('author')->nullable();
    $table->string('source')->nullable();
    $table->enum('status', ['draft', 'published'])->default('published');
    $table->unsignedInteger('view_count')->default(0);
    $table->timestamp('published_at')->nullable();
    $table->timestamps();
});
```

contacts 表：
```php
Schema::create('contacts', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('phone')->nullable();
    $table->string('email')->nullable();
    $table->text('message');
    $table->enum('type', ['联系咨询', '入驻申请'])->default('联系咨询');
    $table->boolean('is_read')->default(false);
    $table->timestamps();
});
```

pages 表（info.html 的静态内容）：
```php
Schema::create('pages', function (Blueprint $table) {
    $table->id();
    $table->string('slug')->unique();              // service, faq, guide, privacy, terms
    $table->string('title');
    $table->longText('content');
    $table->timestamps();
});
```

**Step 1.3 执行迁移**
```bash
php artisan migrate
```

### 阶段 2：数据填充（第 1-2 周交叉）

**Step 2.1 Seed 分类**
```php
// CategorySeeder.php
$categories = [
    ['slug' => 'ciqi',     'name' => '瓷器', 'sort_order' => 1],
    ['slug' => 'yuqi',     'name' => '玉器', 'sort_order' => 2],
    ['slug' => 'shuhua',   'name' => '书画', 'sort_order' => 3],
    ['slug' => 'qianbi',   'name' => '钱币', 'sort_order' => 4],
    ['slug' => 'tongqi',   'name' => '铜器', 'sort_order' => 5],
    ['slug' => 'zaxiang',  'name' => '杂项', 'sort_order' => 6],
    ['slug' => 'wenfang',  'name' => '文房', 'sort_order' => 7],
    ['slug' => 'zhixiu',   'name' => '织绣', 'sort_order' => 8],
];
```

**Step 2.2 Seed 收藏家**
```php
// 从当前 collectors.html 的 6 张卡片数据提取
// avatar 用姓名首字生成（与当前 HTML 一致），或上传真实头像
// slug 用拼音：wang-xiansheng, li-furen, zhang-daoshi, chen-laoshi, liu-xiansheng, zhao-laoshi
```

**Step 2.3 Seed 藏品（40 件）**
```php
// 从当前 collection.html 的 40 张卡片数据提取
// 每件藏品的字段映射：
//   - title: 卡片标题
//   - description: 卡片描述
//   - era: 朝代标签（清代、明代、北宋、战国...）
//   - category_id: 外键关联 categories
//   - collector_id: 外键关联 collectors
//   - cover_image: 当前 assets/collection-*.jpg 的文件名
//   - slug: 从标题生成拼音 slug，手动校验
```

**Step 2.4 Seed 图片迁移**
```bash
# 将当前 assets/ 目录的图片复制到 Laravel storage
cp -r assets/banner-*   public/images/banner/
cp -r assets/collection-* public/images/collection/
cp -r assets/news-*     public/images/news/
cp -r assets/story-*    public/images/story/
cp -r assets/logo.png   public/images/logo.png
```

**Step 2.5 Seed 文章和动态**
```php
// stories: 从 stories.html 的卡片数据提取标题、摘要
//    - content 字段需要手动编写或从占位文本填充
// updates: 从 updates.html 的 24 条卡片数据提取标题、摘要、类型
// pages: 5 条记录（service, faq, guide, privacy, terms）
//         内容从当前 info.html 的各 section 提取
```

### 阶段 3：路由与控制器（第 2 周）

**routes/web.php**
```php
// 前台路由
Route::get('/', [HomeController::class, 'index'])->name('home');

Route::get('/collections', [CollectionController::class, 'index'])->name('collections.index');
Route::get('/collections/{slug}', [CollectionController::class, 'show'])->name('collections.show');

Route::get('/collectors', [CollectorController::class, 'index'])->name('collectors.index');
Route::get('/collectors/{slug}', [CollectorController::class, 'show'])->name('collectors.show');

Route::get('/stories', [StoryController::class, 'index'])->name('stories.index');
Route::get('/stories/{slug}', [StoryController::class, 'show'])->name('stories.show');

Route::get('/updates', [UpdateController::class, 'index'])->name('updates.index');
Route::get('/updates/{slug}', [UpdateController::class, 'show'])->name('updates.show');

Route::get('/contact', [ContactController::class, 'create'])->name('contact.create');
Route::post('/contact', [ContactController::class, 'store'])->name('contact.store');

Route::get('/help', [HelpController::class, 'index'])->name('help.index');
Route::get('/help/{slug}', [HelpController::class, 'show'])->name('help.show');
```

### 阶段 4：Blade 模板迁移（第 2-3 周）

执行顺序（从简单到复杂）：

```
1. app.blade.php 主布局         ← 先搭骨架
2. partials/header.blade.php    ← 第一个组件
3. partials/footer.blade.php    ← 第二个组件
4. partials/ 其他组件
5. help.blade.php               ← 最简单的页面（纯文本内容）
6. contact.blade.php             ← 表单页
7. collectors/index.blade.php   ← 列表页（最简）
8. collectors/detail.blade.php  ← 详情页
9. collection/index.blade.php   ← 列表页（有筛选+搜索+分页）
10. collection/detail.blade.php
11. stories/index.blade.php
12. stories/detail.blade.php
13. updates/index.blade.php
14. updates/detail.blade.php
15. home.blade.php               ← 最后做（依赖其他模块的数据）
```

每完成一个页面后验证：
- 数据正确从数据库渲染
- 分页/筛选/搜索 URL 参数工作正常
- SEO 元数据（title、description）正确
- 图片正确加载

### 阶段 5：阅读量与联系表单（第 3 周）

**阅读量实现：**
```php
// ViewCountService.php
public function increment(string $type, int $id): void
{
    $key = "view_count:{$type}:{$id}";
    $ipKey = "viewed:{$type}:{$id}:" . request()->ip();

    // 5 分钟内同一 IP 不重复计数
    if (Cache::has($ipKey)) {
        return;
    }

    Cache::increment($key);
    Cache::put($ipKey, true, now()->addMinutes(5));
}

// 定时任务同步到数据库（每小时）
// php artisan schedule:run
// ->hourly(function () {
//     同步 Redis/Cache 计数到数据库 view_count 字段
// });
```

**联系表单：**
```php
// ContactController@store
public function store(Request $request)
{
    $validated = $request->validate([
        'name'    => 'required|string|max:50',
        'phone'   => 'nullable|string|max:20',
        'email'   => 'nullable|email|max:100',
        'message' => 'required|string|max:1000',
        'type'    => 'in:联系咨询,入驻申请',
    ]);

    Contact::create($validated);

    return redirect()->back()->with('success', '提交成功，我们会尽快与您联系。');
}
```

### 阶段 6：后台 API（第 4 周，可与前台并行）

Ant Design Pro 后台需要的 API 端点（`routes/api.php`，前缀 `/api/admin`）：

```
GET    /api/admin/dashboard/stats        总览统计
GET    /api/admin/collections            藏品列表（分页、筛选）
POST   /api/admin/collections            创建藏品
PUT    /api/admin/collections/{id}        更新藏品
DELETE /api/admin/collections/{id}        删除藏品
POST   /api/admin/collections/upload      图片上传

GET    /api/admin/collectors             收藏家列表
POST   /api/admin/collectors             创建收藏家
PUT    /api/admin/collectors/{id}/approve 审核通过
PUT    /api/admin/collectors/{id}/reject  拒绝

GET    /api/admin/stories                 故事列表
POST   /api/admin/stories                 创建故事
PUT    /api/admin/stories/{id}           更新故事
DELETE /api/admin/stories/{id}           删除故事

GET    /api/admin/updates                 动态列表
POST   /api/admin/updates                 创建动态
PUT    /api/admin/updates/{id}           更新动态
DELETE /api/admin/updates/{id}           删除动态

GET    /api/admin/pages                   页面列表
PUT    /api/admin/pages/{id}              更新页面内容

GET    /api/admin/contacts                联系消息列表
PUT    /api/admin/contacts/{id}/read       标记已读

GET    /api/admin/categories              分类管理
POST   /api/admin/categories              创建分类
PUT    /api/admin/categories/{id}          更新分类
DELETE /api/admin/categories/{id}          删除分类
```

API 响应统一格式：
```json
{
    "success": true,
    "data": { ... },
    "message": "操作成功"
}
```

API 认证：Laravel Sanctum token，Ant Design Pro 登录后获取 token 放在请求头 `Authorization: Bearer {token}`。

---

## 四、执行检查清单

### 数据库搭建
- [ ] 所有迁移文件创建并执行
- [ ] 模型定义（含关联关系、scope、accessor）
- [ ] Seeder 填充全部演示数据
- [ ] 图片迁移到 public/images/ 目录

### 前台路由与控制器
- [ ] routes/web.php 定义全部前台路由
- [ ] 控制器方法实现（查询、分页、筛选）
- [ ] Blade 模板按顺序完成（15 步）
- [ ] @csrf 表单保护
- [ ] 服务端表单验证
- [ ] 404 页面（未找到 slug 时）

### SEO
- [ ] 每页 <title> 动态生成
- [ ] 每页 <meta description> 动态生成
- [ ] sitemap.xml 生成
- [ ] robots.txt 配置

### 运维
- [ ] 阅读量防重复计数
- [ ] 定时任务（阅读量同步到数据库）
- [ ] 后台 API 认证（Sanctum）
- [ ] 图片上传接口

### 清理
- [ ] 删除当前前端全部 JS 筛选/分页逻辑（由 Laravel 后端处理）
- [ ] 删除 theme.css 中不再需要的样式
- [ ] 删除多余的 tailwind CDN / lucide CDN（改为本地编译或继续 CDN）
- [ ] demo-sidebar.html 临时文件删除
