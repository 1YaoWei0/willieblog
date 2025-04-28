---
title: Flask Turorial 2
categories:
 - flask
comments: true
date: 2025-01-09 20:19:16
description: Flask Turorial 2th
---


# Flask Turorial 2

## Blueprints and Views

一个 view 函数是你写来响应请求的代码。Flask使用模式将传入的请求URL与应该处理它的视图进行匹配。视图返回数据，Flask将这些数据转换为输出响应。Flask也可以走另一个方向，根据视图的名称和参数生成一个URL。

### Create a Blueprint

Blueprint是一种组织一组相关视图和其他代码的方法。它们不是直接在应用程序中注册视图和其他代码，而是在蓝图中注册。然后，当蓝图在工厂功能中可用时，将其注册到应用程序中。

Flaskr将有两个蓝图，一个用于身份验证功能，一个用于博客帖子功能。每个蓝图的代码将放在一个单独的模块中。由于博客需要了解身份验证，因此您将首先编写身份验证。

> flaskr/auth.py

```py
import functools

from flask import (
    Blueprint, flash, g, redirect, render_template, request, session, url_for
)
from werkzeug.security import check_password_hash, generate_password_hash

from flaskr.db import get_db

bp = Blueprint('auth', __name__, url_prefix = '/auth')
```

这将创建一个名为“auth”的蓝图。与应用程序对象一样，蓝图需要知道它的定义位置，因此__name__作为第二个参数传递。url_prefix将被添加到与蓝图关联的所有url之前。

使用app.register_blueprint（）从工厂导入并注册蓝图。在返回应用程序之前，将新代码放在工厂函数的末尾。

> flaskr/__init__.py

```py
    # Register the auth blueprint
    from . import auth
    app.register_blueprint(auth.bp)
```

身份验证蓝图将具有注册新用户以及登录和注销的视图。

### The First View: Register

当用户访问/auth/register URL时，注册视图将返回带有表单的HTML供他们填写。当他们提交表单时，它将验证他们的输入，并再次显示带有错误消息的表单，或者创建新用户并转到登录页面。

现在您只需编写视图代码。在下一页中，您将编写模板来生成HTML表单。

> flaskr/auth.py

```py
@bp.route('/register', methods=('GET', 'POST'))
def register():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        db = get_db()
        error = None

        if not username:
            error = 'Username is required'
        elif not password
            error = 'Password is required'

        if error is None:
            try:
                db.execute(
                    "INSERT INTO user (username, password) VALUES (?, ?)",
                    (username, generate_password_hash(password))
                )
                db.commit()
            except db.IntegrityError:
                error = f"User {username} is already registered"
            else
                return redirect(url_for('auth.login'))
            
        flash(error)

    return render_template('auth/register.html')
```

以下是代码解析：

1. `@bp。route`将URL `/register`与注册视图函数关联起来。当Flask接收到对`/auth/register`的请求时，它将调用register视图并使用返回值作为响应。

2. 如果用户提交了表单，request.method 将是`POST`。在本例中，开始验证输入。

3. `request.form`是一种特殊类型的映射提交表单键和值的字典。用户将输入他们的用户名和密码。

4. 验证 username 和 password 不为空

5. 如果验证成功，插入新的用户数据到数据库  
    - db.execute 使用`?`用于任何用户输入的占位符，以及用于替换占位符的值元组。数据库库将负责转义这些值，这样您就不会受到SQL注入攻击。
    - 为了安全起见，永远不要将密码直接存储在数据库中。相反，使用generate_password_hash（）安全地对密码进行散列，并存储该散列。由于该查询修改了数据，因此需要在之后调用db.commit（）来保存更改。
    - 如果用户名已经存在，则会发生`sqlite3.IntegrityError`，这应该作为另一个验证错误显示给用户。

6. 存储用户之后，它们被重定向到登录页面。`url_for（）`根据登录视图的名称为其生成URL。这比直接编写URL更可取，因为它允许您稍后更改URL，而无需更改链接到它的所有代码。`redirect（）`生成对生成的URL的重定向响应。

7. 如果验证失败，错误将显示给用户。Flash（）存储可以在呈现模板时检索的消息。

8. 当用户最初导航到认证/注册时，或者出现验证错误时，应该显示一个包含注册表单的HTML页面。`render_template（）`将呈现一个包含HTML的模板，该模板将在本教程的下一步中编写。

### Login

```py
@bp.route('/login', methods = ('GET', 'POST'))
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        db = get_db()
        error = None
        user = db.execute(
            'SELECT * FROM user WHERE username = ?', (username)
        ).fetchone()

        if user is None:
            error = 'Incorrect username.'
        elif not check_password_hash(user['password', password]):
            error = 'Incorrect password.'
        
        if error is None:
            session.clear()
            session['user_id'] = user['id']
            return redirect(url_for('index'))
        
        flash(error)

    return render_template('auth/login.html')
```

There are a few differences from the register view:

1. The user is queried first and stored in a variable for later use. `fetchone()` returns one row from the query. If the query returned no results, it returns None. Later, `fetchall()` will be used, which returns a list of all results.

2. `check_password_hash()` hashes the submitted password in the same way as the stored hash and securely compares them. If they match, the password is valid.

3. `session` is a `dict` that stores data across requests. When validation succeeds, the user’s id is stored in a new session. The data is stored in a cookie that is sent to the browser, and the browser then sends it back with subsequent requests. Flask securely signs the data so that it can’t be tampered with.

```py
@bp.before_app_request
def load_logged_in_user():
    user_id = session.get('user_id')

    if user_id is None:
        g.user = None
    else:
        g.user = get_db().execute(
            'SELECT * FROM user WHERE id = ?', (user_id)
        ).fetchone()
```

`bp.before_app_request（）`注册一个在视图函数之前运行的函数，无论请求的URL是什么。`Load_logged_in_user`检查用户id是否存储在会话中，并从数据库中获取该用户的数据，将其存储在g.user上，该数据将持续请求的长度。如果没有用户id，或者用户id不存在，g.user将为None。

### Logout

要注销，需要从会话中删除用户id。那么`load_logged_in_user`将不会在后续请求中加载用户。

```py
@bp.route('/logout')
def logout():
    session.clear()
    return redirect(url_for('index'))
```

### Require Authentication in Other Views

创建、编辑和删除博客文章需要用户登录。可以使用装饰器来检查它应用到的每个视图。

```py
def login_required(view):
    @functools.wraps(view)
    def wrapped_view(**kwargs):
        if g.user is None:
            return redirect(url_for('auth.login'))
        
    return wrapped_view
```

这个装饰器返回一个新的视图函数，该函数包装了它所应用的原始视图。新函数检查用户是否已加载，否则将重定向到登录页面。如果用户被加载，则调用原始视图并正常继续。您将在编写博客视图时使用该装饰器。

### Endpoints and URLs

`url_for（）`函数根据名称和参数生成视图的URL。与视图关联的名称也称为端点，默认情况下，它与视图函数的名称相同。

例如，在教程前面添加到应用程序工厂的`hello（）`视图的名称为‘hello’，可以使用url_for（'hello'）链接到它。如果它接受一个参数，稍后您将看到，它将被链接到使用url_for（'hello', who='World'）。

当使用蓝图时，蓝图的名称被附加到函数的名称之前，因此您在上面编写的登录函数的端点是‘auth ’。Login ‘，因为您将它添加到’auth'蓝图中。

## Templates

模板是包含静态数据和动态数据占位符的文件。使用特定的数据呈现模板以生成最终文档。Flask使用Jinja模板库来呈现模板。

在您的应用程序中，您将使用模板来呈现将显示在用户浏览器中的HTML。在Flask中，Jinja被配置为自动转义HTML模板中呈现的任何数据。这意味着呈现用户输入是安全的；他们输入的任何可能扰乱HTML的字符，如`<and>`，都将被转义为安全值，这些值在浏览器中看起来是一样的，但不会产生不必要的效果。

Jinja的外观和行为都很像Python。使用特殊分隔符将Jinja语法与模板中的静态数据区分开来。`{{and}}`之间的任何内容都是将输出到最终文档的表达式。`{%and%}`表示控制流语句，如if和for。与Python不同，块由开始和结束标记而不是缩进表示，因为块中的静态文本可能会改变缩进。

### The Base Layout

```html
<!doctype html>
<title>{% block title %}{% endblock %} - Flaskr</title>
<link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
<nav>
  <h1>Flaskr</h1>
  <ul>
    {% if g.user %}
      <li><span>{{ g.user['username'] }}</span>
      <li><a href="{{ url_for('auth.logout') }}">Log Out</a>
    {% else %}
      <li><a href="{{ url_for('auth.register') }}">Register</a>
      <li><a href="{{ url_for('auth.login') }}">Log In</a>
    {% endif %}
  </ul>
</nav>
<section class="content">
  <header>
    {% block header %}{% endblock %}
  </header>
  {% for message in get_flashed_messages() %}
    <div class="flash">{{ message }}</div>
  {% endfor %}
  {% block content %}{% endblock %}
</section>
```

### Register

```html
{% extends 'base.html' %}

{% block header %}
  <h1>{% block title %}Register{% endblock %}</h1>
{% endblock %}

{% block content %}
  <form method="post">
    <label for="username">Username</label>
    <input name="username" id="username" required>
    <label for="password">Password</label>
    <input type="password" name="password" id="password" required>
    <input type="submit" value="Register">
  </form>
{% endblock %}
```

### Log In

```html
{% extends 'base.html' %}

{% block header %}
  <h1>{% block title %}Log In{% endblock %}</h1>
{% endblock %}

{% block content %}
  <form method="post">
    <label for="username">Username</label>
    <input name="username" id="username" required>
    <label for="password">Password</label>
    <input type="password" name="password" id="password" required>
    <input type="submit" value="Log In">
  </form>
{% endblock %}
```