---
title: '[Flask] Flaskの基本的な構成（を思い出す）'
date: 2018-06-25 12:13:33
tags:
- python
- flask
---

![flask logo](/images/19-flask.png)

久しぶりに Flask のプロジェクトを作成しようとしたらディレクトリ構成からさっぱり忘れていたので、チュートリアルをなぞって思い出してみる。ついでに最新の環境（pipenv とか）を適用する。

<!-- more -->

## 参考にしたサイト

<a href="https://www.digitalocean.com/community/tutorials/how-to-structure-large-flask-applications" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

基本的にはここの手順通り、ただディレクトリ構成とか少し直している。

## 環境構築

エディタは VSCode を使い、pipenv でモジュール管理を行う。まずはじめに Linter として flake8 と Formatter として autopep8 をインストールする。

```bash
pipenv install flake8 autopep8
```

続いて Flask 関係のモジュールをインストールする。

```bash
pipenv install flask flask-sqlalchemy flask-wtf sqlalchemy
```

### pipenv について

ちょっと余談だけど pipenv は install するとデフォルトで venv 環境（仮想環境）にモジュールがインストールされる。

Python3 組み込みの venv は Windows 環境だと `.venv/Scripts/activate.bat` で環境が有効になり、 `.venv/Scripts/deactivate.bat` で環境から抜ける動きをするはずだが、なぜか deactivate がボクの環境ではうまく動かなく、それが原因で今まで使ってこなかった。（PowerShell を exit すれば仮想環境から抜けれたが、毎回するのはめんどい…）

pipenv では `pipenv shell` で仮想環境が有効になり、 `exit` で抜けることができる。それだけでなく、仮想環境での実行は `pipenv run python ****.py` という風にアクティベートしなくても実行できたり、分かりやすくやり易いようになっている。モジュールのインストールもアクティベート無しで仮想環境の外から行うことができるので、基本的に仮想環境を無効のまま操作することができていい感じだ。

## ディレクトリ構成

先にディレクトリ構成を晒すと以下のような感じにしていく。

```bash
.
├── Pipfile
├── Pipfile.lock
├── README.md
├── app
│   ├── __init__.py
│   ├── controllers
│   │   ├── __init__.py
│   │   └── auth_controller.py
│   ├── forms
│   │   ├── __init__.py
│   │   └── auth_form.py
│   ├── models
│   │   ├── __init__.py
│   │   └── auth.py
│   ├── static
│   └── templates
│       ├── 404.html
│       └── auth
│           └── signin.html
├── config.py
└── run.py
```

## 環境変数の登録

開発環境では以下 2 点の環境変数を設定しておく。

| 環境変数               | 値          |
| ---------------------- | ----------- |
| PIPENV_VENV_IN_PROJECT | true        |
| FLASK_ENV              | development |

## ルートのファイル作成

### run.py

プロジェクトルートに実行の起点となる run.py を作成する。参考サイトでは `host='0.0.0.0'` となっているが、実行したとき [http://0.0.0.0:8000/](http://0.0.0.0:8000/)にアクセスしても正常に表示されなかったので、`host='localhost'` としている。（[http://localhost:8000/](http://localhost:8000/)なら問題なくアクセスできた。)

```py
#!/usr/bin/python3
# coding:UTF-8
""" app/run.py """

from app import app
app.run(host='localhost', port=8080, debug=True)
```

### config.py

flask の設定を記述する、これは参考サイトのままだ。DB は sqlite になっているが、まずはそのままにしておく。

```py
#!/usr/bin/python3
# coding:UTF-8
""" app/config.py """

import os
# Statement for enabling the development environment
DEBUG = True

# Define the application directory
BASE_DIR = os.path.abspath(os.path.dirname(__file__))

# Define the database - we are working with
# SQLite for this example
SQLALCHEMY_DATABASE_URI = 'sqlite:///' + os.path.join(BASE_DIR, 'app.db')
DATABASE_CONNECT_OPTIONS = {}

# Application threads. A common general assumption is
# using 2 per available processor cores - to handle
# incoming requests using one and performing background
# operations using the other.
THREADS_PER_PAGE = 2

# Enable protection agains *Cross-site Request Forgery (CSRF)*
CSRF_ENABLED = True

# Use a secure, unique and absolutely secret key for
# signing the data.
CSRF_SESSION_KEY = "secret"

# Secret key for signing cookies
SECRET_KEY = "secret"
```

## モデルの作成

参考サイトではルートの app ディレクトリ配下にはモジュール単位で機能を置くようになっているが、モジュール単位ではなくレイヤー単位で分けたいと思う。ちなみに django もモジュール単位（機能単位？）で分けるのが慣例のようだ。しかし models.py や controllers.py といったファイルがモジュール別とは言え乱立するのはなんか気持ちが悪いし、rails 的なレイヤー単位のディレクトリ構成に慣れているので後者を採用したい。

```py
#!/usr/bin/python3
# coding:UTF-8
""" app/models/auth.py """

from app import db


class Base(db.Model):

    __abstract__ = True

    id = db.Column(db.Integer, primary_key=True)
    date_created = db.Column(db.DateTime,  default=db.func.current_timestamp())
    date_modified = db.Column(db.DateTime,  default=db.func.current_timestamp(), onupdate=db.func.current_timestamp())


class User(Base):

    __tablename__ = 'auth_user'

    # User Name
    name = db.Column(db.String(128),  nullable=False)

    # Identification Data: email & password
    email = db.Column(db.String(128),  nullable=False,
                      unique=True)
    password = db.Column(db.String(192),  nullable=False)

    # Authorisation Data: role & status
    role = db.Column(db.SmallInteger, nullable=False)
    status = db.Column(db.SmallInteger, nullable=False)

    # New instance instantiation procedure
    def __init__(self, name, email, password):

        self.name = name
        self.email = email
        self.password = password

    def __repr__(self):
        return '<User %r>' % (self.name)
```

### pylint ではなく flake8 を使う

ちなみに Linter として flake8 を採用しているが、flake8 ではなく pylint を使うとこのモデルで大量のエラーを吐く。（`db.Column`が見つからないよ～というエラーが全ての db 参照に出る）

pylint 側の問題なのか、VSCode（Python 拡張）の問題なのか微妙だが、[SQLAlchemy intellisense #292](https://github.com/Microsoft/vscode-python/issues/292)に書いてある通り pylint ではなく flake8 を使うことで一応回避はできる。

## ビューの作成

ビューとしてフォームクラス（`app/forms/auth_form.py`）とテンプレート（`app/templates/auth/signin.html`）を作成する。

```py
#!/usr/bin/python3
# coding:UTF-8
""" app/forms/auth_form.py """

from flask_wtf import Form
from wtforms import TextField, PasswordField
from wtforms.validators import Required, Email


class LoginForm(Form):
    email = TextField('Email Address', [Email(), Required(message='Forgot your email address?')])
    password = PasswordField('Password', [
        Required(message='Must provide a password. ;-)')])
```

```html
{% macro render_field(field, placeholder=None) %} {% if field.errors %}
<div>
  {% elif field.flags.error %}
  <div>
    {% else %}
    <div>
      {% endif %} {% set css_class = 'form-control ' + kwargs.pop('class', '') %} {{ field(class=css_class, placeholder=placeholder,
      **kwargs) }}
    </div>
    {% endmacro %}

    <div>
      <div>
        <legend>Sign in</legend>
        {% with errors = get_flashed_messages(category_filter=["error"]) %} {% if errors %}
        <div>
          {% for error in errors %} {{ error }}
          <br> {% endfor %}
        </div>
        {% endif %} {% endwith %} {% if form.errors %}
        <div>
          {% for field, error in form.errors.items() %} {% for e in error %} {{ e }}
          <br> {% endfor %} {% endfor %}
        </div>
        {% endif %}
        <form method="POST" action="." accept-charset="UTF-8" role="form">
          {{ form.csrf_token }} {{ render_field(form.email, placeholder="Your Email Address", autofocus="") }} {{ render_field(form.password,
          placeholder="Password") }}
          <div>
            <label>
              <input type="checkbox" name="remember" value="1"> Remember Me
            </label>
            <a role="button" href="">Forgot your password?</a>
            <span class="clearfix"></span>
          </div>
          <button type="submit" name="submit">Sign in</button>
        </form>
      </div>
    </div>
```

テンプレートのフォーマットが崩れているのは VSCode のフォーマッタが jinja2 テンプレートを認識していない（サポートされていない）からだ。VSCode 拡張で jinja はあるのだが、コードハイライトしかしてくれないぽい。

正直これだけの理由で jinja2 じゃないテンプレートにしたいんだけど、他のテンプレートエンジンの情報が無さ過ぎて jinja2 から動けないでいる…

## コントローラの作成

基本的にアプリ自体はそれほどスケールしないことがほとんどだが、コントローラが 2 つになったらもう必要になることと、使い方を忘れない為に（！）Blueprint で分割を前提につくる。

```py
#!/usr/bin/python3
# coding:UTF-8
""" app/controllers/auth_controller.py """

from flask import Blueprint, request, render_template, \
    flash, session, redirect, url_for
from werkzeug import check_password_hash
from app.forms.auth_form import LoginForm
from app.models.auth import User

# Define the blueprint: 'auth', set its url prefix: app.url/auth
mod_auth = Blueprint('auth', __name__, url_prefix='/auth')


@mod_auth.route('/signin/', methods=['GET', 'POST'])
def signin():

    # If sign in form is submitted
    form = LoginForm(request.form)

    # Verify the sign in form
    if form.validate_on_submit():

        user = User.query.filter_by(email=form.email.data).first()

        if user and check_password_hash(user.password, form.password.data):

            session['user_id'] = user.id

            flash('Welcome %s' % user.name)

            return redirect(url_for('auth.home'))

        flash('Wrong email or password', 'error-message')

    return render_template("auth/signin.html", form=form)
```

## `app/__init__.py` の作成

最後に `app/__init__.py` を作成し、アプリの初期化や Blueprint へのコントローラ登録を行ったりする。

```py
#!/usr/bin/python3
# coding:UTF-8
""" app/__init__.py """

from flask import Flask, render_template
from flask_sqlalchemy import SQLAlchemy


# Define the WSGI application object
app = Flask(__name__)

# Configurations
app.config.from_object('config')

# Define the database object which is imported
# by modules and controllers
db = SQLAlchemy(app)

# Sample HTTP error handling


@app.errorhandler(404)
def not_found(error):
    return render_template('404.html'), 404


# Import a module / component using its blueprint handler variable (mod_auth)
from app.controllers.auth_controller import mod_auth

# Register blueprint(s)
app.register_blueprint(mod_auth)
# app.register_blueprint(xyz_module)
# ..

# Build the database:
# This will create the database file using SQLAlchemy
db.create_all()
```

## 開発サーバーで動作確認

これで最低限のファイル類は準備できたので、以下コマンドで開発サーバーを立ち上げて動作確認を行う。

```bash
pipenv run python run.py
```

[http://localhost:8000/auth/signin/](http://localhost:8000/auth/signin/)にアクセスしてサインインのフォームが表示されれば OK。

## 実行環境

- Python 3.6.3
- flask 1.0.2
- sqlalchemy 1.2.8
