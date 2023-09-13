# カスタムコマンドについて

Djangoでは`manage.py`よりコマンドを実行できる  
例えば、postというカスタムコマンドを作成した場合

```
python manage.py post
```

上記をコマンドラインで叩けばカスタムコマンドの`handle()`が実行される

また、カスタムコマンドを作成し`cron`等を用いることでバッチ処理を作成できる

カスタムコマンドは`management/commands/`に作成する

## カスタムコマンドの例

```python
# management/command/test_command.py

from django.core.management.base import BaseCommand

class Command(BaseCommand):
    # 'python manage.py test_command help'が実行されたときのメッセージを作成する
    help = "Help message"

    def add_arguments(self, parser):
        # オプション設定時に追記
        pass


    # 実行されるメソッド
    def handle(self, *args, **options):
        print("Hello, World!")
```

## オプション設定
オプションの設定には`add_arguments()`に`parser.add_argument()`を追記する

`parser.add_arguments()`の引数の代表例は以下の通り  
・name_or_flags：コマンドラインから引数に設定する名前  
・nargs：受け取ることができる引数の数  
・default：引数が指定されなかった場合のデフォルト値  
・type：引数の型の指定

実際には`name_or_flags`は省略し、第一引数にオプション名を入れる形が多い
```python
def add_arguments(self, parser):
    # "-name"というオプションを0or1つの引数をとるように設定
    # 指定されなかった場合はNoneが入り、string型を指定
    parser.add_argument("-name", nargs="?", default=None, type=str)
```

<br>

引数名にハイフンをつけない場合は位置引数(順番に応じて呼び出される引数)、つける場合は名前付き引数(名前に応じて呼び出される引数)となる。

```python
# 位置引数となる
parser.add_arguments(name_or_flags="filename")

# 名前付き引数
parser.add_arguments(name_or_flags="-f", "--filename")
```

これらはコマンドラインにて以下のように実行する

```Bash
# 位置引数に値を渡す場合
python manage.py my_command "filename.csv"

# 名前付き引数の場合
python manage.py my_command -f "filename.csv"
python manage.py my_command --filename "filename.csv"
```

<br>

`nargs`の設定は以下のようにする
| 設定文字 | 設定される値 |
|:-----------|:------------|
| * | 0以上 |
| + | 1以上 |
| ? | 0or1 |
| 数字 | 指定した数の引数を強制 |

<br>

## オプションの受け取り
先ほど設定したオプションをhandle()内で受け取るにはget()を使用する

以下のようなカスタムコマンド作成のためのmy_command.pyを作成する
```python
class Command(BaseCommand):
    help = "Help message"

    def add_arguments(self, parser):
        # nameというオプションを作成
        parser.add_argument("-n", "--name")

    def handle(self, *args, **options):
        # optionsとして受け取る
        name = options.get("name", None)
        print(name)
```

このときに以下のようなコマンドラインを実行する
```Bash
python manage.py my_command -n "testName"

# 出力結果
# testName
```

辞書型の`options()`に対してget()を使用することで受け取ることができる

<br>

## 出力について
print()ではなくstdout(),stderr()の仕様が推奨されている
```python
def handle(self, *args, **options):
    self.stdout.write("メッセージ")
    self.stderr.write("エラーメッセージ")

    # 緑色に変更
    self.stdout.write(self.style.SUCCESS("Success"))
```

<br>

## エラーハンドリングについて
エラーハンドリングは`CommandError`を使用
```python
model_id = 100 # 存在しないID
def handle(self, *args, **options):
    try:
        instance = Model.objects.get(pk=model_id)
    except Model.DoesNotExist:
        raise CommandError(f"Model instance does not exists. pk={model_id}")
```


