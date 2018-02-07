# Python全栈之路系列之MySQL SQL注入

`SQL注入`是一种代码注入技术，过去常常用于攻击数据驱动性的应用，比如将恶意的SQL代码注入到特定字段用于实施拖库攻击等。

`SQL注入`的成功必须借助应用程序的安全漏洞，例如用户输入没有经过正确地过滤（针对某些特定字符串）或者没有特别强调类型的时候，都容易造成异常地执行SQL语句。

`SQL注入`是网站渗透中最常用的攻击技术，但是其实SQL注入可以用来攻击所有的SQL数据库。

## SQL注入的实现

创建`SQLdb`数据库

```sql
CREATE DATABASE SQLdb;
```

创建`user_info`表

```sql
CREATE TABLE `user_info` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `username` varchar(32) DEFAULT NULL,
  `password` varchar(32) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

插入一条用户数据

测试的用户名是`ansheng`，密码`as`

```sql
insert into user_info(username,password) values("ansheng","as");
```

Python代码

`app.py`文件

```python
#!/usr/bin/env python
# -*- coding:utf-8 -*-
import tornado.ioloop
import tornado.web
import pymysql

class LoginHandler(tornado.web.RequestHandler):
    def get(self, *args, **kwargs):
        self.render('login.html')
    def post(self, *args, **kwargs):
        username = self.get_argument('username', None)
        pwd = self.get_argument('pwd', None)
        conn = pymysql.connect(host='127.0.0.1', port=3306, user='root', passwd='as', db='sqldb')
        cursor = conn.cursor()
        temp = "select username from user_info where username='%s' and password = '%s'" %(username, pwd,)
        effect_row = cursor.execute(temp)
        result = cursor.fetchone()
        conn.commit()
        cursor.close()
        conn.close()
        if result:
            self.write('登录成功')
        else:
            self.write('登录失败')

application = tornado.web.Application([
    (r"/login", LoginHandler),

])


if __name__ == "__main__":
    application.listen(8888)
    tornado.ioloop.IOLoop.instance().start()
```

HTML代码

`login.html`与`app.py`文件在同级

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<form action="/login" method="post">
    <input type="text" name="username" placeholder="用户名" />
    <input type="text" name="pwd" placeholder="密码" />
    <input type="submit" />
</form>

</body>
</html>
```

演示效果

打开浏览器，输入地址`http://127.0.0.1:8888/login`

填写内容如下：

用户名：`asas ' or 1 = 1-- asd`
密码：`随便填写一串字母`

如图：

![sql-injection-01]../images/2016/12/1483061797.png)

当点击`提交`的时候是否会跳转到登陆成功页面？如果你的代码和我一样，那么就会跳转到`登陆成页面`。

## 为什么出现这种问题？

出现这个问题的主要原因就是因为我们使用了`字符串拼接`的方式来进行SQL指令的拼接。

SQL指令拼接代码

```python
temp = "select username from user_info where username='%s' and password = '%s'" %(username, pwd,)
```

这是一个正常的SQL拼接出来的结果

```sql
select username from user_info where username='ansheng' and password = 'as'
```

这是一个非正常的SQL拼接出来的结果

```sql
select username from user_info where username='asas' or 1 = 1  -- asd' and password = 's'
```

聪明的你是否已经看到其中的玄机了呢？`--`

## 如何防止？

通过`Python`的`pymysql`模块来进行`SQL`的执行，在`pymysql`模块内部会自动把"`'`"(单引号做一个特殊的处理，来预防上述的错误

```python
......
effect_row = cursor.execute("select username from user_info where username=%s and password = %s", (username, pwd))
......
```
