# SQLAlchemy: 反射获取已经定义好的表
当表结构已经定义好了, 或者没有创建表权限时, 定义 ORM 表结构是一件很烦的事情. 但通过 sqlalchemy 的 `reflect` 可以获取到表类型.

```python
# coding: UTF-8

import datetime

from sqlalchemy import create_engine
from sqlalchemy.orm import scoped_session, sessionmaker

engine = create_engine('mysql://username:password@host:port/db?charset=utf8', convert_unicode=True, echo=False)


# 方法一:
from sqlalchemy.ext.declarative import declarative_base
Base = declarative_base()
Base.metadata.reflect(engine)

class Tag(Base):
  __table__ = Base.metadata.tables['tag']

  def to_dict(self):
    d = {}
    d['id'] = self.id
    d['name'] = self.name

    return d


# 方法二:
from sqlalchemy.ext.automap import automap_base
Base2 = automap_base()
Base2.prepare(engine, reflect=True)
Tag2 = Base2.classes.tag

def to_dict(data):
  d = {}
  for column in data._fields:
    value = getattr(data, column)
    if isinstance(value, datetime.datetime):
      d[column] = value.strftime('%Y-%m-%d %H:%M:%S')
    else:
      d[column] = value
  return d

if __name__ == '__main__':
  db_session = scoped_session(sessionmaker(bind=engine))

  for item in db_session.query(Tag):
    print item.to_dict()

  print
  for item in db_session.query(Tag2.id, Tag2.name):
    print to_dict(item)
```

参考链接:
- [Automap](http://docs.sqlalchemy.org/en/latest/orm/extensions/automap.html)
- [How to build a flask application around an already existing database?](http://stackoverflow.com/questions/17652937/how-to-build-a-flask-application-around-an-already-existing-database)

### flask-sqlalchemy:
```python
from flask import Flask
from flask.ext.sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:////tmp/test.db'
db = SQLAlchemy(app)

meta = db.metadata
engine = db.engine

stuffs = Table('stuffs', meta, autoload=True, autoload_with=engine)
```

参考链接: [How can I use SQLAlchemy reflection or metadata with Flask- SQLalchemy?](https://www.reddit.com/r/flask/comments/31etxm/how_can_i_use_sqlalchemy_reflection_or_metadata/)


### other
在第一段代码中有两个 `to_dict` 函数, 分别对应在以下两种情况下如何将查询结果转换为字典:
1. 查询结果有 `Table` 的所有字段, 结果类型为 `Table` 属性时, 类似于 `SELECT * FROM Table`;
2. 查询结果只有几个字段时, 类似于 `SELECT id FROM Table`;

在实际使用中, 以上两种情况都可能会遇到. 所以, 两种 `to_dict` 函数都需要定义.
因此, 只能使用第一种方法获取数据表定义了.

### 改进
当使用第一种方法定义多个数据表时就有点烦了: 因为每张表都要定义一个 `to_dict` 函数.
实际上 python 也是支持多重继承的, 因此, 我们可以定义一个基类并实现 `to_dict` 函数,
然后让所有继承自 `Base` 的数据表子类也继承它.
```python
class BaseModelFunc(object):
  def to_dict(self):
    d = {}
    for column in self.__table__.columns:
      value = getattr(self, column.name)
      if isinstance(value, datetime.datetime):
        value = value.strftime('%Y-%m-%d %H:%M:%S')
      d[column.name] = value

    return d

class Tag(Base, BaseModelFunc):
  __table__ = Base.metadata.tables['tag']
```
