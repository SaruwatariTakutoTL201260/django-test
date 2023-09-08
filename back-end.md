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
        "username" : "testUserName1",
        "email" : "test1@gmail",
        "password" : "testPassword",
        "is_active" : true,
    },
    {
        "username" : "testUserName2",
        "email" : "test2@gmail",
        "password" : "testPassword",
        "is_active" : true,
    },
    {
        "username" : "testUserName3",
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
`AUTHENTICATION_BACKENDS`は認証を上から順に行うため、`email`での認証を行う際には不要な認証を削除する
```python
AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',
    # 'accounts.backends.MyBackend'を削除

    # emailに関する認証を追加
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

class EmailAuthenticationForm(Form):
    email = EmailField(
        label=('Email'), widget=forms.EmailInput(attrs={'autofocus': True,})
    )
    password = forms.CharField(
        label = ("Password"),
        strip = False,
        widget = forms.PasswordInput,
    )
    
    # コンストラクタ
    def _init_(self, request=None, *args, **kwargs):
        self.request = request
        self.user_cache = None
        kwargs.setdefault('label_suffix', '')
        super().__init__(*args, **kwargs)

        # 認証を'email'に変更
        self.email_field = UserModel.__meta.get_field(UserModel.USERNAME_FIELD)
        self.field['email'].max_length = self.email_field.max_length or 254
        if self.fields['email'].label is None:
            # Emailフィールドに変更
            self.fields['email'].label = capfirst(self.email_field.verbose_name)
        for field in self.fields.values():
            field.widget.attrs["class"] = "form_control"
            field.widget.attrs["placeholder"] = field.label

    def clean(self)_:
        email = self.cleaned_data.get('email')
        password = self.cleaned_data.get('password')

        if email is None and password:
            self.user_cache = authenticate(self.request, email=email, password=password)
            if self.user_cache is None:
                raise self.get_invalid_login_error()
            else:
                self.confirm_login_allowd(self.user_cache)

        return self.cleaned_data

    def confirm_login_allowed(self, user):
        if not user.is_active:
            raise forms.ValidationError(
                self.error_messages['inactive'],
                code='inactive'
            )

    def get_user(self):
        return self.user_cache

    def get_invalid_login_error(self):
        return forms.ValidationError(
            self.error_message['invalid_login'],
            code='invalid_login',
            params={'username': _('Email')},
        )
```

このような変更により、`username`と`password`による認証から、`email`と`password`による認証に変更できる。

<br>

# トークン認証について
トークンを用いる場合はDRF(Django REST Framework)を用いるのが一般的

Djangoの標準認証はToken認証に適していないためDRFを使用する

<br>

# DRFでの認証について
## Authの種類について
1,Basic Authentication  
2,Token Authentication  
3,SessionAuthentication  
4,JWT(JSON Web Token)  

<br>

## BasicAuthenticationの例
settings.pyに`BasicAuthentication`を追加
```python
#settings.py

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.BasicAuthentication',
        'rest_framework.authentication.SessionAuthentication',
    ]
}
```

view.pyに変更
```python
# モジュールの追加
from rest_framework.authentication import SessionAuthentication, BasicAuthentication

from rest_framework.permissions import IsAuthenticated

# クラスに必要な情報を追加
class ArticleViewSet(viewsets.ReadOnlyModelViewSet):
    authentication_class = [SessionAuthentication, BasicAuthentication]

    permission_classes = [IsAuthenticated]
```

APIの発信元がHTTPSでないとログイン情報が盗まれるため`Https://`
を使用する

しかし、DjangoのアドミンパネルからログインしていないとAPIが見らないためクライアント側では使えない。

したがって、TokenAuthを使用してみる

<br>

## TokenAuthの使用例
setting.pyの更新
```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication'
    ],

    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated'
    ]
}

INSTALLED_APPS = [
    'rest_framework.authtoken'
]
```


urls.pyにTokenを作成するurlを作成
```python
from rest_framework.authentoken import views

urlpatterns = [
    path('api-token-auth/', views.obtain_auth_token, name='api-token-auth')
]
```

このような設定をし、tokenを使いリクエストするとアクセスが可能


# REMOTE_USERを使用した外部認証
通常の認証ではデータベースに保存されている`User`の情報をもとに検証を行うが、外部認証では`REMOTE_USER`経由で受け取る外部認証ソースにより検証を行う

外部認証ソリューションとして
・IIS + Windows認証
・Apache + mod_authnz_ldap,CAS,Cosign,WebAuth,mod_auth_sspiなどのSSO(シングルサインオン)ソリューションを使用する

<br>

## SSO(シングルサインオン)
各サービスにおけるアクセス時に、その都度認証を行うのが一般的だが合理性に欠けると言われている。  
SSOは連携する全てのサービス間で、一度認証を行うことでアクセスできるようにする仕組みのことである。  
複数のIDとパスワードの管理をなくすことで利便性を向上させるだけでなく、管理者のコストも減らすことが期待できる。

<br>

## `REMOTE_USER`環境変数を設定する

request.META属性で利用可能で、`django.contrib.auth`にある`RemoteUserMiddleware`や`PersistentRemoteUserMiddleware`,`RemoteUserBackend`クラスを使用してREMOTE_USERを使用できる。

<br>

## 使用例

`RemoteUserBackend`というDjangoのREMOTE_USERを操作するためのメソッドがある

`django.contrib.auth.middleware.AuthenticationMiddleware`のある`MIDDLEWARE`に`django.contrib.auth.middleware.RemoteUserMiddleware`を追加する
```python
MIDDLEWARE = [
    'django.contrib.auth.middleware.AuthenticationMiddleware',

    # 以下を追加
    'django.contrib.auth.middleware.RemoteUserMiddleware'
]
```

<br>

さらに`AUTHENTICATION_BACKENDS`の設定で、`ModelBackend`を`RemoteUserBackend`に書き換える
```python
AUTHENTICATION_BACKEND = [
    # 'django.contrib.auth.backends.ModelBackend',を削除

    # 以下を追加
    'django.contrib.auth.backends.RemoteUserBackend',
]
```
この変更により`request.META['REMOTE_USER']`のユーザ名を検証する。  
デフォルトのModelBackendを無効にしたため、`REMOTE_USER`に設定していない値はログインできなくなる。

`RemoteUserBackend`は`ModelBackend`を継承しているため、ModelBackendのパーミッションチェックをそのまま利用できる。  
最たる例として`is_active=False`のデータは認証を許可されない。  

さらなる条件を与える場合は`RemoteUserBackend`を継承した独自のバックエンドを作成する。

<br>

## REMOTE_USERを使う認証の特徴
`メリット`としては  
・外部認証システムを使用可能  
・複数サービス間で認証の共有が可能  
`デメリット`としては  
・外部認証システムの仕様に依存  
・設定やデバッグが難しい  
以上の特徴があると思われる

実際に`REMOTE_USER`を使用するか否かは`SSO`を使用するかで決まる。  
SSOの最大の特徴である複数サービス間での認証の共通化が必要な場合(サービスを複数提供する場合)には`SSO`および`REMOTE_USER`の仕様を検討する
