# 附录1：Msgpack简明教程

msgpack用起来像json，但是却比json快，并且序列化以后的数据长度更小，言外之意，使用msgpack不仅序列化和反序列化的速度快，数据传输量也比json格式小，msgpack同样支持多种语言。

## 安装

msgpack可以使用pip安装，安装命令如下：

```shell
pip install msgpack-python
```

## 使用

### 简单的例子

```python
#coding=utf-8
import datetime
import msgpack
import json

stu = {
    'name':'lili',
    'age':18,
    'score':100
}

#序列化
msg_str = msgpack.packb(stu)
print len(msg_str)
json_str = json.dumps(stu)
print len(json_str)

#反序列化
stu_dict = msgpack.unpackb(msg_str)
print stu_dict
```

程序的运行结果表明，msgpack序列化后的字符串长度为23，而json模块序列化后的字符串长度为41，接近节省了一半的空间。

### 对数据流进行反序列化

msgpack提供了一个Unpacker方法，可以对数据流进行反序列化，下面的代码改自官网的例子

```python
import msgpack
from io import BytesIO

buf = BytesIO()
for i in range(10):
   buf.write(msgpack.packb(range(i)))

buf.seek(0)
print type(buf)
unpacker = msgpack.Unpacker(buf)
for unpacked in unpacker:
    print unpacked
```

### 区分字符串和二进制

json模块，数据经序列化以后，再反序列化，所得到的数据和序列化之前并不完全一致，如果某个数据之前的类型是str，经过反序列化以后，类型就会变成unicode,msgpack提供了一种方法，可以改变这种现状

```python
#coding=utf-8
import datetime
import msgpack
import json

stu = {
    'name':'lili',
    u'age':18,
    'score':100
}

#序列化
msg_str = msgpack.packb(stu,use_bin_type=True)
json_str = json.dumps(stu)

#反序列化
print msgpack.unpackb(msg_str,encoding='utf-8')
print json.loads(json_str)
```

最后的输出结果如下

```
{u'age': 18, 'score': 100, 'name': 'lili'}
{u'age': 18, u'score': 100, u'name': u'lili'}
```

### 自定义类型数据的序列化

msgpack序列化函数提供了一个default参数，反序列化函数提供了一个object\_hook，其用法，与上一篇json中的default和objec\_thook一样

```python
#coding=utf-8
import datetime
import msgpack
import json

stu = {
    'name':'lili',
    u'age':18,
    'score':100,
    "date": datetime.datetime.now()
}

def decode_datetime(obj):
    if b'__datetime__' in obj:
        obj = datetime.datetime.strptime(obj["as_str"], "%Y-%m-%d%H:%M:%S")
    return obj

def encode_datetime(obj):
    if isinstance(obj, datetime.datetime):
        return {'__datetime__': True, 'as_str': obj.strftime("%Y-%m-%d%H:%M:%S")}
    return obj

packed_dict = msgpack.packb(stu, default=encode_datetime)
this_dict_again = msgpack.unpackb(packed_dict, object_hook=decode_datetime)
print this_dict_again
```



