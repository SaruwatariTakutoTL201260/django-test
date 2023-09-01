# バックエンド認証

## 設定方法
Djangoには`authentication backends`という認証のためのリストがある

このリストに設定するにはsetting.pyに`AUTHENTICATION_BACKENDS`に書き込む

デフォルトでは`django.contrib.auth.backends.ModelBackend`が設定してある

`AUTHENTICATION_BACKEND`に記載してある順に`authenticate()`を呼び出し、認証に成功すればユーザーを返す

## 認証バックエンドをカスタマイズ
`accounts/backends.py`を作成しバックエンド認証を追加する
引数は`get_user(user_id)`と`authenticate(request,**credentials)`である
```python
# accounts/backends.py

from django.contrib.auth.backends import ModelBackend
from django.contrib.auth.models import User

class MyBackend(ModelBackend):
    def authenticate(self, request, username=None, password=None)
    # ユーザー名/パスワードを確認し、ユーザーを返す
    try:
        user = User.objects.get(username=username)
    # userがなければ'None'を返す
    except User.DoesNotExist:
        return None
    else:
        # パスワードの一致とuserの認証をチェック
        if user.check_password(password) and self.user_can_authenticate(user):
            return user
```
Userの`name`と`password`の両方が一致すれば、ユーザーを返す

一致しなければ`None`を返す


<br>

## バックエンドを追加
作成したバックエンドを`AUTHENTICATION_BACKENDS`に追加
```python
AUTHENTICATION_BACKEND = [
    'django.contrib.auth.backends.ModelBackend',
    'accounts.backends.MyBackend',
]
```

<br>

## ログインフォームを作成し、認証する
AuthenticationFormはDjangoに標準で用意されているフォームであり、ユーザ名とパスワードの認証用のフォーム

```python
# accounts/forms.py
from django import forms
from django.contrib.auth import authenticate
from django.contrib.auth.forms import AuthenticationForm
from django.contrib.auth.models import User

class CustomLoginForm(forms.Form):
    username = forms.CharField(label='Username')
    # forms.widgetはパスワード入力用ウィジェット
    password = forms.CharField(label='Password', widget=forms.PasswordInput)

    def clean(self):
        # clean()を呼ぶための処理
        clean_data = super().clean()
        username = cleaned_data.get('username')
        password = cleaned_data.get('password')

        if username and password:
            # 認証処理
            user = authenticate(username=username, password=password)
            if user is None:
                raise forms.ValidationError("ユーザ名またはパスワードが正しくありません")
            elif not user.is_active:
                raise forms.ValidationError("アカウントが無効です")

        # usernameとpasswordを返す
        return self.cleaned_data
```

<br>

viewに変更を加えてCustomLoginFormを使用できるようにする
```python
# accounts/view.py

from django.contrib.auth.views import LoginView
from .forms import CustomLoginForm

class CustomLoginView(LoginView):
    # CustomLoginFormを追加
    form_class = CustomLoginForm
```

<br>

`urls.py`にloginのpathを追記
```python
# accounts/urls.py

urlpatterns = [
    path(`login/`, views.CustomLoginView.as_view(), name='login'),
]
```

<br>

# ユーザ(User)について
ユーザ認証を作成するにはユーザ(User)モデルについて理解する必要がある。
ユーザモデルクラスの仕様には以下のようにインポートする
```python
from django.contrib.auth.models import User

# カスタマイズしたユーザモデル
from django.contrib.auth import get_user_model
User = get_user_model()
```

デフォルトのユーザモデルは下記のフィールドを持つ

`username`  
`first_name`  
`last_name`  
`email`  
`password`
`is_staff`  
`is_active`  
`is_superuser`  
`last_login`  
`date_joined`

# 具体例
デフォルトのUserにデータを与える
```json:seed.json
{
    {
        "username" : testUserName1,
        "email" : "test1@gmail",
        "password" : "testPassword",
        "is_active" : true,
    },
    {
        "username" : testUserName2,
        "email" : "test2@gmail",
        "password" : "testPassword",
        "is_active" : true,
    },
    {
        "username" : testUserName3,
        "email" : "test3@gmail",
        "password" : "testPassword",
        "is_active" : false,
    },
}
```

## 取得してみる
login画面にて以下の通りにユーザ名とパスワードを入れてみる
|| ユーザ名(username) | パスワード(password) |
|:-|:-----------|:------------|
|1| testUserName1 | testPassword |
|2| testUserName2| MissPassword |
|3| testUserName3 | testPassword |

<br>

ログイン結果は下記の通り
|  | 出力結果 |
|:-|:-|
|1|成功|
|2|ValidationError("ユーザ名またはパスワードが正しくありません")|
|3|ValidationError("アカウントが無効です")|
3に関しては`is_active`が`false`なため認証に失敗している

<br>

# メール認証にする
'username'による認証から'email'の認証に変更する

## フォームの追加
```python
# accounts/form.py

class EmailAuthBackend(ModelBackend):
    def authenticate(self, request, email=None, password=None):
        try:
            user = User.objects.get(email=email)
        except User.DoesNotExist:
            return None
        else:
            if user.check_password(password) and self.user_can_authenticate(user):
                return user
```

<br>

## AUTHENTICATION_BACKENDSに追加
```python
AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',
    'accounts.backends.MyBackend',
    'accounts.backends.EmailAuthBackend'
]
```

## formsを変更
```python
from django.contrib.auth import authenticate
from django.forms import ModelForm, Form
from django.forms.fields import EmailField
from django.import forms
from django.contrib.auth.forms import AuthenticationForm
from django.contrib.auth.models import User

class 

```
