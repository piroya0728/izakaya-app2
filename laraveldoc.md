# Laravel 初心者向けDoc

#### 環境構築(docker, laravel-sail) は飛ばします。

<br>

## 1. Laravelの基本構造

### MVCの基本構造について


```
ルーティングファイル　→　/routes/web.php
モデル　→　/app/Models/...
ビュー　→　/resources/views/...
コントローラー　→　/app/Http/Controllers/...
```

### 1-1. ルーティングの指定方法
```
// デフォルトではLaravelのトップページが表示される
Route::get('/', function () {
    return view('index');
});
``` 

アクションを指定する場合(こっちがメイン)
```
// 指定方法テンプレ
Route::method('PATH', [[Controller Name]::class, '[Action Name]'])

// 実装例(nameメソッドを使用すると、()内の引数をリダイレクトなどの際の引数に定できる。任意。後述。)
Route::get('/admin/blogs', [AdminBlogController::class, 'index'])->name('admin.blogs.index')
```

### 1-2. ビューファイル
ブレードテンプレートという形式のファイルを使用する。
```
// ファイル名を下記のように指定するだけで作成できる。
/views/○○○.blade.php
// これを作ったら後は普通にHTMLを書くだけ。

// 使用方法 view関数を使用
return view('○○○', array());
// 第一引数を指定する際、'blade.php'は省略する！
// array() => php内の変数をビュー側で使う際は、この中に変数名をキーとした連想配列を定義する。第二引数は省略可。

// 記述例
return view('admin.blogs.index', ['blogs' => $blogs, 'user' => $user]);
```

ブレードテンプレートの特徴
```
// {{ $○○○ }}でエスケープできる。
// ↑と↓は同義
<?php echo htmlspecialchars($○○○); ?>

// {!! $○○○ !!}でエスケープせずに出力。

// (-- ○○○ --)でコメント。
(-- このテキストはコメントとなり出力されません --)

Bladeディレクティブ
→PHPの処理がもっと簡単に書ける@から始まるBladeテンプレートの特別な構文

// 例1　<?php ?>は…
@php
    // PHPの処理;
@endphp
処理内容がきちんと分けられていれば、↑を使うことはあまりないかも

他にも、、、
@if, @elseif, @else, endif
@while, @for, @foreach, @forslseディレクティブもある

一覧は下記リンクを参照
https://readouble.com/laravel/8.x/ja/blade.html
```

### 1-3. コントローラーファイル

作成コマンド
```
sail artisan make:controller EventController
```
このファイルの中に関数を記述していく
注)　ルーティングでアクションを指定する際は、ルーティングファイル上部でuse文を使い、コントローラーをインポートする必要がある。
```
// アクションを指定するために、↓use文でインポート
use App\Http\Controllers\Admin\UserController;

Route::get('/admin/users/create', [UserController::class, 'create'])->name('admin.users.create');
```

### 1-4. リクエスト
#### 1-4-1. 送信データの取得方法

```
// タイプヒンティング(メソッドの引数のデータ型を指定すること)
// 指定したデータ型が入ってしまうというミスを防ぐことができる。
// Requestと指定したら、フォームの入力値が勝手に入る。変数名はタイプヒンティング後に記述。

例) <input type="text" name="keyword">の入力値を取得する場合
public function queryStrings(Request $request)
{
    $keyword = $request->keyword;
    return 'キーワードは'. $keyword. 'です。';
}
```

#### 1-4-2. csrfディレクティブ

GETメソッド以外で送信する場合、セキュリティ対策のためにcsrfトークンも送信が必要。
Laravelではcsrf対策を自動で行うため、フォーム内に「@csrf」と記述する必要がある。
csrfトークンがないと、不正なアクセスとみなし、処理を実行せず419のエラーページに飛ばす。
```
例) ブログ投稿
<form action="{{ route('admin.blogs.store') }}" method='POST' enctype="multipart/form-data">
    @csrf
    <h3>ブログ登録</h3>
    <button type="submit">登録</button>
    // ↓入力エリア

</form>
```

#### 1-4-3. HTTPリクエストメソッド


| メソッド | 役割 |
|:---|:---|
|GET |データの取得(通常のリンク遷移) |
|POST |データの新規登録、GETで対応できないような秘匿性のあるもの、容量が大きいもの |
|PUT |データの上書き |
|PATCH |データの部分更新 |
|DELETE |データの削除 |

GETとPOSTだけでも問題ないが、URLの乱立を防いだりルート管理がしやすくなるため、リクエストメソッドは使い分けた方が良い。

ちなみに、
HTMLのフォームはGETとPOSTしかサポートしていない。

Laravelでは@methodディレクティブを使うことで、疑似的にGET、POST以外のメソッドに対応できる。

```
例) ブログの更新
<form action="{{ route('admin.blogs.update', ['blog' => $blog->id]) }}" method='POST' enctype="multipart/form-data">
    @csrf
    @method('PUT')
    <h3>ブログ編集</h3>
    <button type="submit">更新</button>
    // ↓入力エリア

</form>
```

#### 1-4-4. リソースルート(典型的なルートの一括登録)
1つのコンテンツに対して、下記の7つのアクションが必要になることが多い。

| アクション | 役割 |
|:---|:---|
|index |一覧表示用 |
|show |詳細表示用 |
|create |登録フォーム表示用 |
|store |登録処理用 |
|edit |更新フォーム表示用 |
|update |更新処理用 |
|destroy |削除処理用 |

1つ1つ書いていたらルートを7つも作ることになってしまう。

↓ resourceメソッドで一括管理し、ルートの登録を省略
```
Route::resource('/events', EventController::class);

// 7ルート必要ではない場合、指定したものだけを設定するonly、指定したものだけを除外するexceptメソッドを使用する。
Route::resource('/events', EventController::class)->only('index', 'create', 'store'); // 一覧表示と登録のみ
Route::resource('/events', EventController::class)->except('edit', 'update', 'destroy'); // 一覧表示と詳細表示、登録のみ

// 作成方法
// コントローラー作成時に--resource または -rオプションを追加
// 下記2つは同義
sail artisan make:controller EventController --resource
sail artisan make:controller EventController -r
```










