---
title: Flask Tutorial 1
tags:
 - flask
categories:
 - flask
comments: true
date: 2025-01-08 19:23:28
---


# Flask Tutorial

## Project Layout

创建一个项目文件夹然后打开。

```powershell
mkdir flask-tutorial # 创建 flask 项目文件夹
cd flask-tutorial
```

按照 [Flask Overview](https://1yaowei0.github.io/2025/01/07/ai-plateform-flask/) 设置项目目录。

***

一个大的**flask**项目，不可以将所有的代码放到一个文件。**Python**项目使用*package*将代码组织成多个模块，这些模块可以在需要的地方导入。

这个项目应该包含：

- `flaskr/`, 一个包含应用代码和文件的 python 包；
- `tests/`, 一个包含测试模块的目录；
- `.venv/`, 一个包含 flask 和其它已安装依赖的虚拟环境；
- 一个告诉 python 如何安装项目的安装文件；
- 版本控制配置，如 git。无论项目大小，你都应该养成对所有项目使用某种版本控制的习惯；
- 如何可能在未来添加到项目的文件。

最后，一个项目的结构应该如下：

```txt
/home/user/Projects/flask-tutorial
├── flaskr/
│   ├── __init__.py
│   ├── db.py
│   ├── schema.sql
│   ├── auth.py
│   ├── blog.py
│   ├── templates/
│   │   ├── base.html
│   │   ├── auth/
│   │   │   ├── login.html
│   │   │   └── register.html
│   │   └── blog/
│   │       ├── create.html
│   │       ├── index.html
│   │       └── update.html
│   └── static/
│       └── style.css
├── tests/
│   ├── conftest.py
│   ├── data.sql
│   ├── test_factory.py
│   ├── test_db.py
│   ├── test_auth.py
│   └── test_blog.py
├── .venv/
├── pyproject.toml
└── MANIFEST.in
```

如果使用版本控制，可以如下设置。下列被指定的路径将会被版本控制忽略，即使它在项目的开发过程中被修改。

```txt
.gitignore
.venv/

*.pyc
__pycache__/

instance/

.pytest_cache/
.coverage
htmlcov/

dist/
build/
*.egg-info/
```

## Application Setup

一个 Flask 应用是一个 Flask 类的实例。任何与该应用相关的配置，URLs 都会在这个类中被注册。

创建 Flask 引用的最直截了当的方式是在你代码的目录顶部创建一个全局的 Flask 实例。

你将在一个函数中创建它，而不是全局地创建 Flask 的引用（实例）。这个函数称为 *应用工厂*。任何配置，注册和其它应用所需的设置都需要在这个函数中发生，然后返回该应用。

### The Application Factory

在 flaskr 目录下添加 `__init__.py` 文件，它有双重的职责，它将包含应用工厂，以及它告诉 python 这个 flaskr 目录应该被当作一个包。

```py
import os

from flask import Flask

def create_app(test_config = None):
    # Create and configue the app
    app = Flask(__name__, instance_relative_config = True)
    app.config.from_mapping(
        SECRET_KEY = 'dev',
        DATABASE = os.path.join(app.instance_path, 'flaskr.sqlite')
    )

    if test_config is None:
        # Load the instance config, if it exists, when not testing
        app.config.from_pyfile('config.py', silent = True)
    else:
        # Load the test config if passed in
        app.config.from_mapping(test_config)

    # ensure the instance folder exists
    try:
        os.makedirs(app.instance_path)
    except OSError:
        pass

    # a simple page that says hello
    @app.route('/hello')
    def hello():
        return 'Hello, World!'
    
    return app
```

`create_app` 是一个应用工厂函数。你将在教程后面添加它，但它已经做了很多事儿。

1. `app = Flask(__name__, instance_relative_config = True)` 创建了 Flask 实例。
    - `__name__` 是当前 python 模块的名字。这个 app 需要知道它的位置来设置一些路径。并且 `__name__` 是一种很方便告诉它的方式；
    - `instance_relative_config` 告诉 app 配置文件是相对于实例文件夹的。实例文件夹位于flaskr包之外，可以保存不应该提交给版本控制的本地数据，例如配置秘密和数据库文件。

2. `app.config.from_mapping()` 设置了 app 的一些默认配置：
    - `SECRET_KEY` 被用来保证数据安全。它被设置为‘dev’，以便在开发期间提供一个方便的值，但在部署时应该使用随机值覆盖它；
    - `DATABASE` 是 SQLite 数据库将被保存的位置。它在 `app.instance_path` 之下，这是Flask为实例文件夹选择的路径。

3. `app.config.from_pyfile()` 使用从实例文件夹中的config.py文件（如果存在）获取的值覆盖默认配置。例如，在部署时，可以使用它来设置一个真正的SECRET_KEY。
    - `Test_config` 也可以传递给工厂，并代替实例配置使用。

4. `os.makedir` 确保 `app.instance_path` 存在，Flask 不会自动创建实例文件夹，但是它需要被创建因为当前的项目将创建数据库文件。

5. `app.route` 创建了一个简单的路由，在深入后面教程之前我们可以看见应用是工作的。它创建了一个在 URL 和返回了响应的应用之间的连接。

## Run The Application

现在可以使用flask命令运行应用程序了。在终端中，告诉Flask在哪里可以找到您的应用程序，然后在调试模式下运行它。记住，您仍然应该在顶层的flask-tutorial目录中，而不是在flask包中。

调试模式在页面引发异常时显示交互式调试器，并在更改代码时重新启动服务器。您可以让它继续运行，只要按照本教程重新加载浏览器页面即可。

```powershell
flask --app flaskr run --debug

* Serving Flask app "flaskr"
* Debug mode: on
* Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
* Restarting with stat
* Debugger is active!
* Debugger PIN: nnn-nnn-nnn
```

## Define and Access the Database

SQLite 很方便因为它不需要设置一个独立的数据库服务，它是在 python 中编译的。但是，如果并发请求试图同时写入数据库，它们将会变慢，因为每次写入都是顺序进行的。小型应用程序不会注意到这一点。当您的企业规模扩大后，您可能希望切换到不同的数据库。

### Connect to the Database

使用 SQLite 数据库（以及大多数其他Python数据库库）时要做的第一件事是创建到它的连接。任何查询和操作都使用连接执行，该连接在工作完成后关闭。

在 web 应用程序中，这种连接通常与请求绑定在一起。它是在处理请求时创建的，并在发送响应之前关闭。

> flaskr/db.py

```py
import sqlite3
from datetime import datetime

import click
from flask import current_app, g


def get_db():
    if 'db' not in g:
        g.db = sqlite3.connect(
            current_app.config['DATABASE'],
            detect_types=sqlite3.PARSE_DECLTYPES
        )
        g.db.row_factory = sqlite3.Row

    return g.db


def close_db(e=None):
    db = g.pop('db', None)

    if db is not None:
        db.close()
```

**g** 是一个对于每个请求都唯一的对象。它用于存储在请求期间可能被多个函数访问的数据。如果在同一请求中第二次调用get_db，则存储并重用该连接，而不是创建新连接。

**current_app** 是另一个特殊对象，指向处理请求的Flask应用程序。由于使用了应用程序工厂，因此在编写其余代码时没有应用程序对象。Get_db将在创建应用程序并处理请求时调用，因此可以使用current_app。

**sqlite3.connect()** 建立到DATABASE配置键所指向的文件的连接。这个文件并不一定要存在，直到稍后初始化数据库时才会存在。

**sqlite3.Row** 告诉连接返回行为类似字典的行。这允许按名称访问列。

**close_db** 通过检查是否设置了g.db来检查是否创建了连接。如果连接存在，则关闭该连接。接下来，您将告诉应用程序有关应用程序工厂中的close_db函数，以便在每次请求之后调用它。

### Create the Tables

```sql
DROP TABLE IF EXISTS user;
DROP TABLE IF EXISTS post;

CREATE TABLE user (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  username TEXT UNIQUE NOT NULL,
  password TEXT NOT NULL
);

CREATE TABLE post (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  author_id INTEGER NOT NULL,
  created TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  title TEXT NOT NULL,
  body TEXT NOT NULL,
  FOREIGN KEY (author_id) REFERENCES user (id)
);
```

往 `db.py` 放入 python 方法用于运行 SQL 命令。

```py
def init_db():
    db = get_db()

    with current_app.open_resource('schema.sql') as f:
        db.executescript(f.read().decode('utf8'))

@click.command('init-db')
def init_db_command():
    init_db()
    click.echo('Initialized the database.')

sqlite3.register_converter(
    "timestamp", lambda v: datetime.fromisoformat(v.decode())
)
```

`Open_resource（）`打开一个相对于flask包的文件，这很有用，因为在以后部署应用程序时，您不一定知道该位置在哪里。Get_db返回一个数据库连接，用于执行从文件中读取的命令。

`command（）`定义了一个名为init-db的命令行命令，该命令调用init_db函数并向用户显示成功消息。您可以阅读命令行接口来了解更多关于编写命令的信息。

调用`sqlite3.register_converter（）`告诉Python如何解释数据库中的时间戳值。我们将该值转换为datetime.datetime。

### Register with the Application

`close_db`和`init_db_command`函数需要在应用实例中注册；否则，它们将不会被应用程序使用。但是，由于使用的是工厂函数，因此在编写函数时该实例不可用。相反，应该编写一个函数，它接受一个应用程序并进行注册。

```py
def init_app(app):
    app.teardown_appcontext(close_db)
    app.cli.add_command(init_db_command)
```

`app.teardown_appcontext（）`告诉Flask在返回响应后进行清理时调用该函数。

`App.cli.add_command（）`添加了一个可以通过flask命令调用的新命令。

从工厂导入并调用这个函数。在返回应用程序之前，将新代码放在工厂函数的末尾。

```py
    # a simple page that says hello
    @app.route('/hello/')
    def hello():
        return 'Hello, World!'
```

### Initialize the Database File

现在init-db已经注册到应用程序中，可以使用flask命令调用它，类似于上一页中的run命令。

> 如果您仍在运行上一页中的服务器，您可以停止服务器，或者在新的终端中运行此命令。如果您使用一个新的终端，请记住切换到您的项目目录并激活env，如安装中所述。

运行`init-db`命令：

```powershell
flask --app flaskr init-db
```

将会有一个 `flaskr.sqlite` 文件出现在项目的`instance`目录中