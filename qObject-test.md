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