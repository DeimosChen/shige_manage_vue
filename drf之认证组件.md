# drf之认证组件

drf中提供了组件帮我们实现认证功能，通过认证的用户可以访问视图，不通过的不能访问。下面介绍drf中认证组件的写法和如何快速配置

## 认证使用

1. 写一个类，继承BaseAuthentication（其实不继承也可以，只是继承这个类规定我们必须重写authenticate）

2. **重写authenticate方法**，内部写校验逻辑（获取token，从数据库中取，比对）

3. 校验成功，**返回：user对象，token**。返回中第一个必须放user对象。

	校验失败，抛出异常，使用APIException或者AuthenticationFailed

	如果需要使用多个认证类，那么返回两个值的认证类一定要放在配置认证类的列表的最后

4. 全局使用：setting中配置

	局部使用：视图中写authentication_classes =[]

	局部禁用：空列表

## 源码分析

图片来自：

![](C:\Users\Administrator\Desktop\微信图片_20200710150341.png)

核心是Request中的 `_authenticate(self):`

## 快速配置

这里使用用户名，密码和token来进行认证，登陆成功后返回一个token，同时将token存进数据库，token表与用户表为一对一关系，首先写模型类

```python
# models.py

from django.db import models
class User(models.Model):
# 用户表
    username = models.CharField(max_length=32)
    password = models.CharField(max_length=32)
    user_type = models.IntegerField(choices=((1, '管理员'), (2, '一般用户'), (3, 'vip用户')))
    # 数字对应后面的中文说明，通过get_字段名_display()就能取出choice后面的中文

class User_Token(models.Model):
    # token表，token用uuid生成，长一些
    token = models.CharField(max_length=128)
    user = models.OneToOneField(to=User, on_delete=models.DO_NOTHING)
    # 与用户表做一对一外键
```

在应用下新建一个py文件，专门用来放认证组件的代码

```python
# myauth.py

from rest_framework.authentication import BaseAuthentication
from rest_framework.exceptions import AuthenticationFailed
from app01.models import User_Token

class LoginAuthenticate(BaseAuthentication):
# 认证类，继承BaseAuthentication，其中重写authenticate方法
    def authenticate(self, request):
        token = request.GET.get('token')
        # 认证的逻辑按照实际情况来，这里假设从get请求的请求体中拿token
        if token:
            user_token = User_Token.objects.filter(token=token).first()
            # token成功获取，去token表中获取用户对象
            if user_token:
            # token与数据库中的一致，校验成功，就可以通过外键字段获取用户对象
                return user_token.user,token
                # 返回用户对象，token，两个顺序不能反过来
            else:
                raise AuthenticationFailed('认证失败')
        else:
        # 校验不成功，没有token或者token不存在，使用这个模块来抛出异常
            raise AuthenticationFailed('没有token，需要登陆')
```

下面把认证类放到视图中使用

```python
from rest_framework.views import APIView
from rest_framework.viewsets import ModelViewSet,ViewSetMixin
import uuid # 用于生成token
from app01 import models
from app01 import ser
from rest_framework.response import Response
from rest_framework.decorators import action # ModelViewSet要用的装饰器
from app01.myauth import LoginAuthenticate # 自己写的认证模块

class LoginView(APIView):
# 登陆视图，不用配置认证
    def post(self,request):
        username = request.data.get('username')
        password = request.data.get('password')
        user_obj = models.User.objects.filter(username=username,password=password).first()
        print(request.data)
        # 对用户名和密码验证，成功就给你一个token
        if user_obj:
            token = uuid.uuid4()
            print(token)
            models.User_Token.objects.update_or_create(defaults={'token':token},user=user_obj)
            # 使用update_or_create，在User_Token表中按照user查找是否存在，然后根据token去新建或更新值
            return Response({"msg":"ok"})
        return Response({"msg":"failed"})
    
class BookViewSet(ModelViewSet):
    # 局部使用认证组件：authentication_classes，列表中写导入的认证类，可以写多个
    authentication_classes = [LoginAuthenticate]
    queryset = models.Book.objects.all()
    serializer_class = ser.BookSerializer

    @action(methods=['get'],detail=False) # ModelViewSet的装饰器，get请求的时候触发视图，如果没有登陆，认证组件中就会抛出异常
    def bookAction(self,request):
        print(request.user.username)
        book_obj = self.get_queryset()[:3]
        ser = self.get_serializer(book_obj,many=True)
        # 自己写逻辑
        return Response(ser.data)
    # 只是简单的返回Response，可以自己封装response
```

全局配置认证

```python
# settings.py
REST_FRAMEWORK={
    "DEFAULT_AUTHENTICATION_CLASSES":["app01.service.auth.Authentication",]
}
```

# 自定义权限组件

用户登陆之后，可以根据用户类型：管理员，VIP，普通用户，给用户分发权限。权限组件可以和认证组件配合使用

## 源码简析

```python
# APIView---->dispatch---->initial--->self.check_permissions(request)(APIView的对象方法)
def check_permissions(self, request):
	# 遍历权限对象列表得到一个个权限对象(权限器)，进行权限认证
	for permission in self.get_permissions():
	# 权限类一定有一个has_permission权限方法，用来做权限认证的
	# 参数：权限对象self、请求对象request、视图类对象
	# 返回值：有权限返回True，无权限返回False
		if not permission.has_permission(request, self):
			self.permission_denied(
			request, message=getattr(permission, 'message', None)
			)
```

## 快速使用

1. 定义一个权限类，继承BasePermission

2. 重写has_permission方法，接收request，view

3. 因为权限配置在认证之后，所以user已经登陆，可以在user里面获取用户的信息（用户类别）

4. has_permission中写逻辑，有权限返回True，无权限返回False

5. 视图中使用权限组件，可以全局配置也可以局部配置

	permission_classes = [app_auth.UserPermission]

	REST_FRAMEWORK={
	    "DEFAULT_AUTHENTICATION_CLASSES":["app01.app_auth.MyAuthentication",],
	    'DEFAULT_PERMISSION_CLASSES': [
	        'app01.app_auth.UserPermission',
	    ],

```python
# permission.py

from rest_framework.permissions import BasePermission
# 导入BasePermission
class MyPermission(BasePermission):
# 写权限类，继承BasePermission，内部重写has_permission
    def has_permission(self, request, view):
        user = request.user
        # 已经登陆，可以直接从request里面获取用户对象
        print(user.get_user_type_display())
        # 可以拿到choices里面的中文字段
        if user.user_type == 1:
        # 自己写权限的逻辑（只有用户类型1的管理员可以访问这个视图）
            return True
        else:
            return False
```

```python
# views.py

class BookViewSet(ModelViewSet):
    authentication_classes = [LoginAuthenticate]
    permission_classes = [MyPermission]
    # 权限写在认证的下面
    queryset = models.Book.objects.all()
    serializer_class = ser.BookSerializer

    @action(methods=['get'],detail=False)
    def bookAction(self,request):
        ...
        return Response(返回的数据)
```

局部使用，就像上面那样，在认证类下面写permission_classes = [MyPermission]，可以有多个权限类，执行顺序从左到右

全局使用，去setting.py里面写

```python
REST_FRAMEWORK={
    "DEFAULT_AUTHENTICATION_CLASSES":["自己写的认证类",],
    'DEFAULT_PERMISSION_CLASSES': [
        '自己写的权限类',
    ],
}
```

