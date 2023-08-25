# Qオブジェクトの使用方法
Qをインポートすることで使用可能となる。
```
from django.db.models import Q
```
<br>

`Q(フィールド名__条件=検索したいワード)`が基本構文である。
```example
Q(name_exact=test)
```

条件に関しては以下の表を参照。
|条件名      | 条件内容 |
|:-----------|:------------|
|incontains  |大文字小文字関係なく含まれている|
| contains   | 含まれている |
|exact       |完全一致|
|startswith  |はじめとの一致|
|endswith    |終わりとの一致|
|gte         |以上|
|lte         |以下|

filter()と組み合わせて`OR`や`not`の条件をクエリを生成できる。

# Qオブジェクトの使用例
object.filter()に`OR`の条件をセット
```
or_object = Article.objects.filter(
    Q(num__gte = 20) | Q(id__exact == 2)
)
```
<br>

object.filter()に`and`の条件をセット
```
and_object = Article.objects.filter(Q(name__contains = 'test') & Q(age__gte = 5))
```

<br>

object.filter()に`not`の条件をセット
```
not_object = Article.objects.filter(~Q(name__contains == 'test')).all()
```

<br>

# 具体例
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
from django.db.models import Q
from datetime import datetime

# 特定の日付を取得
target_date = datetime(2021,1,1)

# target_dateの日付より後または、is_deleted=trueのデータをクエリに設定する。
query = (
    Q(is_deleted__exact=true) | Q(modified__gt=target_date)
)

articles = Article.objects.filter(query).all()
```

取り出されたデータは以下の通りになる
```json
{
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
・filter()は複数条件が並ぶ際に`and`で結合する仕様となっている。
Qオブジェクトを使用することで`OR`や`NOT`の条件を満たすことができる。

# デメリット
・クエリセットのためのコードが長くなるため、複雑なクエリになると可読性が下がる。

・Qオブジェクトの前に純粋なlookup関数を使用すると正しくクエリが生成されない。

*lookup関数とは
```python
# この形をとるDjangoのfilterメソッドの基本形
QuerySet.objects.filter(フィールド名__lookup="value")
```

<br>

NG例
```python
#Qが後に来ているので、NG
Poll.objects.get(
    question__startswith='Who',
    Q(pub_date=date(2005, 5, 2)) | Q(pub_date=date(2005, 5, 6))
)
```

正しい例
```python
#Qが先に来ているので、OK
Poll.objects.get(
    Q(pub_date=date(2005, 5, 2)) | Q(pub_date=date(2005, 5, 6)),
    question__startswith='Who',
)
```
