---
layout : life
title: python 库学习之 itsdangerous
category : Python学习
duoshuo: true
date : 2016-03-14
---

[这是原始主页](http://pythonhosted.org/itsdangerous/)

<!-- more -->

有时候你想向一个不可信的环境发送数据，但是如何安全的完成这个任务呢？诀窍就是引入签名！使用只有你自己知道的秘钥进行加密，并把加密后的数据发给别人处理，当你取回数据时，你就可以确保没人篡改过这份数据；

虽然接受者可以解密你的数据看看里面封装的是什么内容，但他们没法修改你的内容，除非他们知道你的私钥，因此你只要保管好你的秘钥，同时使用复杂的秘钥就万事大吉了；

itsdangerous内部默认使用了HMAC和SHA1来签名，基于 Django 签名模块。它也支持JSON Web 签名 (JWS)。这个库采用BSD协议，由Armin Ronacher编写，而大部分设计与实现的版权归Simon Willison和其他的把这个库变为现实的Django爱好者们；

## 1. 安装

你可以从 `PyPI` 上直接安装这个库：

```
    pip install itsdangerous
```

**适用案例：**

* 在取消订阅某个通讯时，你可以在URL里序列化并且签名一个用户的ID。这种情况下你不需要生成一个一次性的token并把它们存到数据库中, 同样适用于一些激活链接等类似的情景下；
* 被签名的对象可以被保存在例如 `cookie` 等不可信来源，这意味着你不需要在服务端保存 `session` , 这样可以降低数据库读取次数；
* 通常签名后的数据可以安全的往返与服务端和客户端之间，这个特性可以用于将服务端的状态传递到客户端在传递回来；

## 2. 签名接口

最基本的接口是签名接口， `Signer` 类可以用来将一个签名附加到指定 string 上：

```
    from itsdangerous import Signer
    s = Signer('secret-key')
    s.sign('my string')
    'my string.wh6tMHxLgJqB6oY1uT73iMlyrOA'
```

签名会被加在字符串尾部，中间由句号 (.)分隔。验证字符串，使用 unsign() 方法：

```
    s.unsign('my string.wh6tMHxLgJqB6oY1uT73iMlyrOA')
    'my string'
```

如果被签名的是一个unicode字符串，那么它将隐式地被转换成utf-8。然而，在反签名时，你没法知道它原来是unicode还是字节串。

如果反签名失败了，将得到一个异常：

```
    s.unsign('my string.wh6tMHxLgJqB6oY1uT73iMlyrOX')
    
    Traceback (most recent call last):
      ...
    itsdangerous.BadSignature: Signature "wh6tMHxLgJqB6oY1uT73iMlyrOX" does not match
```

## 3. 使用时间戳签名

如果你想要一个可以过期的签名，可以使用 `TimestampSigner` 类，它会加入时间戳信息并签名。在反签名时，你可以验证时间戳就没有过期；

```
    from itsdangerous import TimestampSigner
    s = TimestampSigner('secret-key')
    string = s.sign('foo')
    # 参数的意思为在签名时间后的多少秒为有效
    s.unsign(string, max_age=5)
    Traceback (most recent call last):
        ...
    itsdangerous.SignatureExpired: Signature age 15 > 5 seconds
```

## 4. 序列化

因为字符串难以处理，本模块也提供了一个与json或pickle类似的序列化接口.(它内部默认使用simplejson，但是可以通过子类进行修改),这个 `Serializer` 类实现了:

```
    from itsdangerous import Serializer
    s = Serializer('secret-key')
    s.dumps([1, 2, 3, 4])
    '[1, 2, 3, 4].r7R9RhGgDPvvWl3iNzLuIIfELmo'
```

它当然也可以加载数据:

```
    s.loads('[1, 2, 3, 4].r7R9RhGgDPvvWl3iNzLuIIfELmo')
    [1, 2, 3, 4]
```

若果你需要一个时间戳， 你可以使用 `TimedSerializer` 类

## 5. URL安全序列化

如果只能向一个字符受限的环境中传递可信的字符串的话，这个功能将会十分有用。`itsdangerous` 提供了一个 `URL` 安全序列化工具:

```
    from itsdangerous import URLSafeSerializer
    s = URLSafeSerializer('secret-key')
    s.dumps([1, 2, 3, 4])
    'WzEsMiwzLDRd.wSPHqC0gR7VUqivlSukJ0IeTDgo'
    
    s.loads('WzEsMiwzLDRd.wSPHqC0gR7VUqivlSukJ0IeTDgo')
    [1, 2, 3, 4]
```

## 6. JSON WEB 签名

从“itsdangerous” 0.18版本开始，也支持了JSON Web签名。它们的工作方式与原有的URL安全序列化器差不多，但是会根据当前JSON Web签名（JWS）草案（10） [draft-ietf-jose-json-web-signature] 来生成header。

```
    from itsdangerous import JSONWebSignatureSerializer
    s = JSONWebSignatureSerializer('secret-key')
    s.dumps({'x': 42})
    'eyJhbGciOiJIUzI1NiJ9.eyJ4Ijo0Mn0.ZdTn1YyGz9Yx5B5wNpWRL221G1WpVE5fPCPKNuc6UAo'
```

在将值加载回来时，默认会像其他序列化器一样，不会返回header。但是你可以通过传入 return_header=True 参数来得到header，定制的header会在序列化的数据外层:

```
    s.dumps(0, header_fields={'v': 1})
    'eyJhbGciOiJIUzI1NiIsInYiOjF9.MA.wT-RZI9YU06R919VBdAfTLn82_iIQD70J_j-3F4z_aM'
    
    s.loads('eyJhbGciOiJIUzI1NiIsInYiOjF9.MA.wT-RZI9YU06R919VBdAfTLn82_iIQD70J_j-3F4z_aM', return_header=True)
    (0, {u'alg': u'HS256', u'v': 1})
```

itsdangerous目前只提供HMAC SHA的派生算法以及不使用算法，不支持基于ECC的算法。header中的算法将与序列化器中的进行核对，如果不匹配，将引发 BadSignature 异常。

## 7. 盐

所有的类都接受一个盐的参数。这个名字可能会误导你，因为通常你会认为，密码学中的盐会是和被签名的字符串存储在一起的东西，用来防止彩虹表查找，这种盐是公开的。

与Django中的原始实现类似，itsdangerous中的盐，是为了一个截然不同的目的而产生的。你可以将它视为成命名空间。就算你泄露了它，也不是很严重的问题，因为没有密钥的话，它对攻击者没什么帮助。

假设你想签名两个链接。你的系统有个激活链接，用来激活一个用户账户，并且你有一个升级链接，可以让一个用户账户升级为付费用户，这两个链接使用email发送。在这两种情况下，如果你签名的都是用户ID，那么该用户可以在激活账户和升级账户时，复用URL的可变部分。现在你可以在你签名的地方加上更多信息（如升级或激活的意图），但是你也可以用不同的盐：

```
    s1 = URLSafeSerializer('secret', salt='activate-salt')
    s1.dumps(42)
    'NDI.kubVFOOugP5PAIfEqLJbXQbfTxs'
    
    s2 = URLSafeSerializer('secret-key', salt='upgrade-salt')
    s2.dumps(42)
    'NDI.7lx-N1P-z2veJ7nT1_2bnTkjGTE'
    
    s2.loads(s1.dumps(42))
    Traceback (most recent call last):
      ...
    itsdangerous.BadSignature: Signature "kubVFOOugP5PAIfEqLJbXQbfTxs" does not match
```

只有使用相同盐的序列化器才能成功把值加载出来：

```
    s2.loads(s2.dumps(42))
    42
```

## 8. 对失败的响应

从itsdangerous 0.14版本开始，异常会有一些有用的属性，可以允许你在签名检查失败时，检查你的数据。这里必须极其小心，因为这个时候，你知道有某人修改了你的数据。但这可能对你debug很有帮助。

示例用法:

```
    from itsdangerous import URLSafeSerializer, BadSignature, BadData
    s = URLSafeSerializer('secret-key')
    decoded_payload = None
    try:
        decoded_payload = s.loads(data)
        # This payload is decoded and safe
    except BadSignature, e:
        encoded_payload = e.payload
        if encoded_payload is not None:
            try:
                decoded_payload = s.load_payload(encoded_payload)
            except BadData:
                pass
            # 这里的数据可以解码出来，但不是安全的，因为有某人改动了签名。
            # 解码步骤(load_payload)是显式的，因为将数据反序列化可能是不安全的
            #（请设想被解码的不是json,而是pickle）
```

如果你不想检查到底是哪里出错了，你也可以使用不安全的加载方式:

```
    from itsdangerous import URLSafeSerializer
    s = URLSafeSerializer('secret-key')
    sig_okay, payload = s.loads_unsafe(data)
```

返回的元组中第一项是一个布尔值，表明了签名是否是正确的。

## 9. Python 3 提示

在Python 3中，itsdangerous的接口在一开始可能让人困扰。由于它封装在内部的序列化器，函数返回值不一定是unicode字符串还是字节对象。内置的签名器总是基于字节的。

这是为了允许模块操作不同的序列化器，独立于它们的实现方式。模块通过执行一个空对象的序列化，来决定使用哪种序列化器。

## 10. It’s Dangerous 的升级日志

[直接看英文版本](http://pythonhosted.org/itsdangerous/#it-s-dangerous-changelog)
    
    

