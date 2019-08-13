---
layout: post
title: "04--Django数据库配置与原生SQL"
date: 2019-08-16
tags: Django  
---

##### @Author  : Roger TX (425144880@qq.com)  
##### @Link    : https://github.com/paotong999  

# 一、Django数据库模型
## 1.1 Django模型数据库配置
Django 模型是与数据库相关的，与数据库相关的代码一般写在 models.py 中，Django 支持 sqlite3，MySQL，Oracle等数据库，需要在settings.py中配置数据源。

Mysql 连接配置：
```
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'autotest', #数据库名称
        'USER': 'root', #用户名
        'PASSWORD': '123456',  # 密码
        'HOST': '192.168.199.1',  # HOST
        'PORT': '3306',  # 端口
    }
}
```

Oracle 连接配置：
```
service_name
DATABASES = {
　　'default': {
　　　　'ENGINE': 'django.db.backends.oracle',
　　　　'NAME': 'IP:端口号/service_name',
　　　　'USER': '用户名',
　　　　'PASSWORD': '密码',
　　}
}
```

## 1.2 Mysql 数据库依赖包
在使用Mysql时，需要安装PyMySQL。安装之后，在setting中配置完数据库后启动django

```
django.core.exceptions.ImproperlyConfigured: Error loading MySQLdb module.
Did you install mysqlclient?
```

如果出现上面错误，需要在 `project_name` 的 `__init__.py` 中加上

```
import pymysql
pymysql.install_as_MySQLdb()
```

之后再次启动django

```
  File "D:\github\txenv\lib\site-packages\django\db\backends\mysql\base.py", line 36, in <module>
    raise ImproperlyConfigured('mysqlclient 1.3.13 or newer is required; you have %s.' % Database.__version__)
django.core.exceptions.ImproperlyConfigured: mysqlclient 1.3.13 or newer is required; you have 0.9.3.
```

进入 `base.py` 后找到36行，这里判断，如果version < 1.3.13直接抛出异常，需要注释掉这两行

```
#if version < (1, 3, 13):
#    raise ImproperlyConfigured('mysqlclient 1.3.13 or newer is required; you have %s.' % Database.__version__)
```

之后再次启动django

```
  File "D:\github\txenv\lib\site-packages\django\db\backends\mysql\operations.py", line 146, in last_executed_query
    query = query.decode(errors='replace')
AttributeError: 'str' object has no attribute 'decode'
```

进入 `operations.py` 后找到146行，将 `decode` 改为 `encode`

```
        if query is not None:
#            query = query.decode(errors='replace')
            query = query.encode(errors='replace')
```

## 1.3 Oracle 数据库依赖包
在使用Oracle时，需要安装cx_Oracle。但是要注意以下几点：
1. cx_Oracle的位数一定要与python还有instantclient位数一致，要么都是32位，要么都是64位
2. 将oci.dll，oraocci11.dll，oraociei11.dll，拷贝到Python目录下
3. 第二步行不通的话，将instantclient_11_2的目录写进环境变量

***

# 二、Django中使用原生Sql
## 一：extra:结果集修改器，一种提供额外查询参数的机制
```
from django.db import models

class Book(models.Model):
    name = models.CharField('书名')
    price = models.IntegerField（'价钱'）
    publish = models.CharField('出版社')
    create_time = models.DateTimeField（'上线日期'）
```
> - books= Book.objects.filter(publish='清华出版社').extra(where=['price>50'])
> - models.Book.objects.filter(publish='人民出版社', price=50)
> - models.Book.objects.extra(select={'count': 'select count(*) from Book'})

## 二：raw:执行原始sql并返回模型实例
raw() 自动将查询中的字段映射到模型上的字段。
> books = Book.objects.raw('select * from book where publish="清华出版社"')

或者，您可以使用translations参数to 将查询中的字段映射到模型字段 raw()。这是一个字典，将查询中字段的名称映射到模型上字段的名称。例如，上面的查询也可以写成：
> - name_map = {'user_name': 'name', 'book_price': 'price', 'pls': 'publish', 'time': 'create_time'}
> - Book.objects.raw('select * from book', translations=name_map)

如果需要参数化的查询，可以向raw()方法传递params参数。
> books = Book.objects.raw('select * from book where name = %s', [lname])

## 三：直接执行自定义Sql
**游标和连接**
```
# 建立默认数据库连接的游标 
from django.db import connection
cursor=connection.cursor()    # with connection.cursor() as cursor

# 获取指定数据库的连接和游标
from django.db import connections
with connections['my_db_alias'].cursor() as cursor:
```
将游标作为上下文管理器使用:
```
with connection.cursor() as c:
    c.execute(...)
```
等价于：
```
c = connection.cursor()
try:
    c.execute(...)
finally:
    c.close()
```

**Python DB API下规范下cursor对象常用接口：**
1. description：如果cursor执行了查询的sql代码。那么读取cursor.description属性的时候，将返回一个列表，这个列表中装的是元组，元组中装的分别是(name,type_code,display_size,internal_size,precision,scale,null_ok)，其中name代表的是查找出来的数据的字段名称，其他参数暂时用处不大。
2. rowcount：代表的是在执行了sql语句后受影响的行数。
3. close：关闭游标。关闭游标以后就再也不能使用了，否则会抛出异常。
4. execute(sql[,parameters])：执行某个sql语句。如果在执行sql语句的时候还需要传递参数，那么可以传给parameters参数。如果查询中包含百分号字符，需要写成两个百分号字符：
> `cursor.execute("SELECT foo FROM bar WHERE baz = '30%%' AND id = %s", [self.id])`

5. fetchone：在执行了查询操作以后，获取第一条数据。
6. fetchmany(size)：在执行查询操作以后，获取多条数据。具体是多少条要看传的size参数。如果不传size参数，那么默认是获取第一条数据。
7. fetchall：获取所有满足sql语句的数据。

Python DB API会返回不带字段的结果，这意味着你得到的是一个列表，而不是一个字典。
```
def dictfetchall(cursor):
    "Returns all rows from a cursor as a dict"
    desc = cursor.description
    return [
        dict(zip([col[0] for col in desc], row))
        for row in cursor.fetchall()
    ]
```