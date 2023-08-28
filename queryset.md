# クエリセットとは
Djangoによる、Queryset型のオブジェクトのリスト
モデルからデータを取り出すことができる
呼び出すメソッドにより、様々な形でデータを取得可能

<br>

# 使用方法
## すべてのデータを取得
```python
queryset = Article.objects.all()
```

## 自身のキーを指定して取得
```python
queryset = Article.objects.get(pk=1)
```

## 取得条件を指定する
```python
queryset = Article.objects.filter(name="test")
```

## 複数条件をAND指定して取得
```python
queryset = Article.objects.filter(name="test",is_deleted=1)
```

## 複数条件をOR指定して取得
```python
from django.db.models import Q
queryset = Article.objects.filter(Q(name="test") | Q(contents__startswith="test"))
```

## 大小や範囲指定も可能
```python
# age>=2
queryset = Article.objects.filter(age__gte=2)

# 5<=age<=10
queryset = Article.objects.filter(age__range=(5,10))
```

## ソート
```python
# 昇順
queryset = Article.objects.order_by("name")

# 降順
queryset = Article.objects.filter(name="test").order_by("-name")
```

<br>

# 使用例
## model構成
```python:model.py
from django.db import models
from django.utils import timezone

class Article(models.Model):
    name = models.TextField()
    contents = models.TextField()
    is_deleted = models.BooleanField(default=False)
    modified = models.DateTimeField(default=timezone.now)
```

## DB
先ほどのモデルに以下のようにデータが入っていたとする
```json:seed.json
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
```python:view.py
# is_deleted=trueであり、modifiedを対象とした降順
queryset = Article.objects.filter(is_deleted='true').order_by("-modified")
```

結果は以下
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

# メリット
複雑なクエリを簡単に書くことができる

# デメリット
複雑なクエリを書くときは可読性に欠ける