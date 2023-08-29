# マネージャー
マネージャーとは、Djangoのデータベースクエリの操作を提供するインターフェース

## マネージャーの名前
デフォルトでは`objects`という名前の`Manage`を各モデルクラスに追加

クラス上でobjects以外の名前をつけることも可能
```python
from django.db import models
# Articleモデルを想定
class Article(models.Model)
    article = models.Manager()
```

<br>

# マネージャーのカスタマイズ
特定のモデル用にカスタマイズしたManagerを使うことが可能

マネージャーのカスタマイズは以下の2つ

・マネージャーに新しいメソッドを追加

・マネージャーが初めに返すQuerysetを修正

<br>

# 使用例
## マネージャーに新しいメソッドを追加
```python
from django.db import models

class ArticleManager(models.Manager):
    def active(self):
        return self.filter(is_deleted=True)

class Article(models.Model):
    is_deleted = models.BooleanField(default=False)

    # デフォルトのマネージャにメソッドを追加
    objects = ArticleManager()
```

作成したメソッドを使用することで以下のように簡潔にクエリを作成できる

```python
# is_deleted=True
queryset = Article.objects.active()

# active()を使用しない場合
queryset = Article.objects.filter(is_deleted=True)
```
さらに複雑なクエリになる場合、メソッドを追加することで簡潔に書くことができる

<br>

以下のように特定のモデルに対するマネージャの書き方もある

```python
from django.contrib.auth.models import UserManager as BaseUserManager
from django.contrib.auth.models import AbstractUser
 
class UserManager(BaseUserManager):
 
    def active(self):
        """アクティブユーザーを取得"""
        return self.filter(is_active=True)
 
class User(AbstractUser):
    objects = UserManager()
```

<br>

# 複数のメソッドを使用

`as_manager()`を使用することで複数のメソッドを使用することができる

```python
class ItemQuerySet(models.QuerySet):
    def on_sale(self) -> QuerySet:
        queryset = self.get_queryset()
        return queryset.filter(released_at__lte=datetime.now())
    
    def female(self):
        queryset = self.get_queryset()
        return queryset.filter(category__name='female')

class Item(Models.Model):
    name = models.CharField(min_length=3, max_length=127)
    price = models.DecimalField(max_digits=17, decimal_places=2)
    released_at = models.DateTimeField(null=True, blank=True)
    category = models.ForeignKey(
        Category,
        on_delete=models.CASCADE
    )

    objects = ItemQuerySet.as_manager()
```

通常のmanagerは複数のメソッドを重ねて使うことができない

`as_manager`を使用することで、managerをQuerySetメソッド(filter()やall()と同義)と扱うため、重ねて使用できる

```python
# 複数のメソッドを重ねて利用
Item.objects.on_sale().female()
```

<br>

## マネージャーが初めに返すQuerysetを修正
```python
from django.db import models

class DahlBookManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(author='Roald Dahl')

class Book(models.Model):
    title = models.CharField(max_length=100)
    objects = models.Manager()
    dahl_objects = DahlBookManager()
```

このManagerにより、クエリセットを修正することが可能


```python
#デフォルトのマネージャ
queryset = Book.objects.all()
queryset = Book.dahl_objects.all()
```

Managerを複数定義することで取得するデータを複数パターン変更可能

<br>

# 具体例
## model構成
```python
from django.db import models
from django.utils import timezone

class ArticleManager(models.Model):
    def active(self):
        # is_deleted = Trueのデータのみ取得
        return self.filter(is_deleted=True)

class Article(models.Model):
    name = models.TextField()
    contents = models.TextField()
    is_deleted = models.BooleanField(default=False)
    modified = models.DateTimeField(default=timezone.now)

    # ArticleManagerをデフォルトに設定する
    objects.ArticleManager()
```

## DB
先ほどのモデルに以下のようにデータが入っていたとする
```json
{
    {
        "id" : 1,
        "contents" : "testContents1",
        "is_deleted" : true,
        "modified" : "2020-12-21 12:00:00",
    },
    {
        "id" : 2,
        "contents" : "testContents2",
        "is_deleted" : true,
        "modified" : "2021-03-22 3:32:26"
    },
    {
        "id" : 3,
        "contents" : "testContents3",
        "is_deleted" : false,
        "modified" : "2022-09-03 18:54:03"
    },
}
```

## 取得してみる
```python
# Article.Manager()のactive()が呼び出される
articles = Article.objects.active().all()
```

取り出されたデータは以下の通りになる
```json
{
    {
        "id" : 1,
        "contents" : "testContents1",
        "is_deleted" : true,
        "modified" : "2020-12-21 12:00:00",
    },
    {
        "id" : 2,
        "contents" : "testContents2",
        "is_deleted" : true,
        "modified" : "2021-03-22 3:32:26"
    },
}
```

<br>

# querysetとの関係性
マネージャーを使用することでquerysetの処理を共通化することができる。

例えば、以下のような`IsDeletedManager()`を全てのテーブルに`is_deleted = IsDeletedManager()`とすることで、削除されていないデータを取得することができる

```python
class IsDeletedManager(models.Manager):
    def active(self):
        return self.filter(is_deleted=False)
```


## QuerySetとManagerの違い

・ManagerはModelのクエリ操作を提供するインターフェース

・querysetはManagerManagerから呼び出されるオブジェクトを表す

そのため、querysetはview.pyにて処理のたびにクエリを書き込む必要があるが、マネージャーを使用することで、処理の共通化やviewの可読性の向上につながる

<br>

# querysetのカスタマイズ
QuerySetのサブクラスを作成することで、QuerySetも同様にカスタマイズすることが可能
サブクラスの定義には`django.db.models.query.QuerySet`を継承する
## カスタムメソッドを追加
```python
# querysets.py
from django.db.models.query import QuerySet

# サブクラス
class QuerySetByIsDeleted(QuerySet):
    def notDeleted(self):
        # 削除されていないデータのみ取得
        return self.filter(is_deleted=false)
```
これをモデルに追加することでカスタムメソッドを使用可能

```python
#model.py

from django.db import models
from myapp.querysets import QuerySetByIsDeleted

class ArticleManager(models.Manager):
    def get_queryset(self):
        return QuerySetByIsDeleted(self.model, using=self._db)

class Article(models.Model):
    name = models.CharField(max_length=255)
    is_deleted = models.BooleanField(default=False)
    exist = models.ArticleManager()
    objects = models.Manager()
```

追加した`exist()`はクエリセットとして使用することができる
```python
# view.py

# is_deleted = Trueのデータを全件取得
queryset = Article.exist.all()
```