# Django Advanced Interview Questions

## Table of Contents

### Django REST Framework
- [Q1: What is Django REST Framework and why use it?](#q1)
- [Q2: How do serializers work in DRF?](#q2)
- [Q3: Explain ViewSets and Routers in DRF](#q3)
- [Q4: How do you handle authentication in DRF?](#q4)
- [Q5: How do you implement pagination in DRF?](#q5)

### Authentication & Authorization
- [Q6: How does Django's authentication system work?](#q6)
- [Q7: How do you implement custom user models?](#q7)
- [Q8: What are permissions and how do you use them?](#q8)

### Signals & Async
- [Q9: What are Django signals?](#q9)
- [Q10: How do you use async views in Django?](#q10)

### Performance & Caching
- [Q11: How do you optimize Django QuerySets?](#q11)
- [Q12: How does caching work in Django?](#q12)

### Deployment & Security
- [Q13: What security features does Django provide?](#q13)
- [Q14: How do you deploy Django applications?](#q14)
- [Q15: What is Celery and how do you use it with Django?](#q15)

---

## Django REST Framework

<a id="q1"></a>
### Q1: What is Django REST Framework and why use it?
**Answer:**

Django REST Framework (DRF) is a powerful toolkit for building Web APIs in Django.

**Key features:**
- Serialization (complex data ↔ JSON/XML)
- Authentication & permissions
- Browsable API
- Pagination
- Throttling
- ViewSets & routers

```python
# Installation
# pip install djangorestframework

# settings.py
INSTALLED_APPS = [
    # ...
    'rest_framework',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.TokenAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
}

# Basic API view
from rest_framework.decorators import api_view
from rest_framework.response import Response

@api_view(['GET', 'POST'])
def article_list(request):
    if request.method == 'GET':
        articles = Article.objects.all()
        serializer = ArticleSerializer(articles, many=True)
        return Response(serializer.data)
    
    elif request.method == 'POST':
        serializer = ArticleSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(author=request.user)
            return Response(serializer.data, status=201)
        return Response(serializer.errors, status=400)

# Class-based API view
from rest_framework.views import APIView
from rest_framework import status

class ArticleList(APIView):
    def get(self, request):
        articles = Article.objects.all()
        serializer = ArticleSerializer(articles, many=True)
        return Response(serializer.data)
    
    def post(self, request):
        serializer = ArticleSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(author=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

<a id="q2"></a>
### Q2: How do serializers work in DRF?
**Answer:**

Serializers convert complex data (QuerySets, model instances) to Python datatypes that can be rendered to JSON/XML.

```python
from rest_framework import serializers
from .models import Article, Category, Tag

# Basic Serializer
class ArticleSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(max_length=200)
    content = serializers.CharField()
    created_at = serializers.DateTimeField(read_only=True)
    
    def create(self, validated_data):
        return Article.objects.create(**validated_data)
    
    def update(self, instance, validated_data):
        instance.title = validated_data.get('title', instance.title)
        instance.content = validated_data.get('content', instance.content)
        instance.save()
        return instance

# ModelSerializer (more common)
class ArticleSerializer(serializers.ModelSerializer):
    class Meta:
        model = Article
        fields = ['id', 'title', 'content', 'author', 'created_at']
        read_only_fields = ['author', 'created_at']
    
    # Custom field validation
    def validate_title(self, value):
        if len(value) < 5:
            raise serializers.ValidationError("Title must be at least 5 characters")
        return value
    
    # Object-level validation
    def validate(self, data):
        if data.get('title') and data.get('content'):
            if data['title'] in data['content']:
                raise serializers.ValidationError("Content should not repeat title")
        return data

# Nested serializers
class CategorySerializer(serializers.ModelSerializer):
    class Meta:
        model = Category
        fields = ['id', 'name', 'slug']

class ArticleDetailSerializer(serializers.ModelSerializer):
    category = CategorySerializer(read_only=True)
    category_id = serializers.PrimaryKeyRelatedField(
        queryset=Category.objects.all(),
        source='category',
        write_only=True
    )
    tags = serializers.StringRelatedField(many=True, read_only=True)
    author = serializers.SerializerMethodField()
    
    class Meta:
        model = Article
        fields = ['id', 'title', 'content', 'category', 'category_id', 'tags', 'author']
    
    def get_author(self, obj):
        return {
            'id': obj.author.id,
            'username': obj.author.username,
            'email': obj.author.email
        }

# Writable nested serializers
class ArticleCreateSerializer(serializers.ModelSerializer):
    tags = serializers.ListField(
        child=serializers.CharField(),
        write_only=True
    )
    
    class Meta:
        model = Article
        fields = ['title', 'content', 'category', 'tags']
    
    def create(self, validated_data):
        tags_data = validated_data.pop('tags', [])
        article = Article.objects.create(**validated_data)
        
        for tag_name in tags_data:
            tag, _ = Tag.objects.get_or_create(name=tag_name)
            article.tags.add(tag)
        
        return article

# Usage
serializer = ArticleSerializer(article)  # Serialize
serializer.data  # {'id': 1, 'title': '...', ...}

serializer = ArticleSerializer(data=request.data)  # Deserialize
serializer.is_valid(raise_exception=True)
serializer.save(author=request.user)
```

<a id="q3"></a>
### Q3: Explain ViewSets and Routers in DRF
**Answer:**

ViewSets combine related views into a single class. Routers automatically generate URL patterns.

```python
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import Article
from .serializers import ArticleSerializer

# ModelViewSet - full CRUD
class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    lookup_field = 'slug'  # Use slug instead of pk
    
    def get_queryset(self):
        queryset = Article.objects.all()
        status = self.request.query_params.get('status')
        if status:
            queryset = queryset.filter(status=status)
        return queryset
    
    def get_serializer_class(self):
        if self.action == 'list':
            return ArticleListSerializer
        return ArticleDetailSerializer
    
    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
    
    # Custom actions
    @action(detail=True, methods=['post'])
    def publish(self, request, slug=None):
        article = self.get_object()
        article.publish()
        return Response({'status': 'published'})
    
    @action(detail=False, methods=['get'])
    def recent(self, request):
        recent = Article.objects.order_by('-created_at')[:5]
        serializer = self.get_serializer(recent, many=True)
        return Response(serializer.data)
    
    @action(detail=True, methods=['post', 'delete'])
    def favorite(self, request, slug=None):
        article = self.get_object()
        if request.method == 'POST':
            request.user.favorites.add(article)
            return Response({'status': 'favorited'})
        else:
            request.user.favorites.remove(article)
            return Response({'status': 'unfavorited'})

# ReadOnlyModelViewSet - list and retrieve only
class CategoryViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Category.objects.all()
    serializer_class = CategorySerializer

# Router configuration
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'articles', ArticleViewSet, basename='article')
router.register(r'categories', CategoryViewSet, basename='category')

# urls.py
from django.urls import path, include

urlpatterns = [
    path('api/', include(router.urls)),
]

# Generated URLs:
# GET    /api/articles/         - list
# POST   /api/articles/         - create
# GET    /api/articles/{slug}/  - retrieve
# PUT    /api/articles/{slug}/  - update
# PATCH  /api/articles/{slug}/  - partial_update
# DELETE /api/articles/{slug}/  - destroy
# POST   /api/articles/{slug}/publish/  - custom action
# GET    /api/articles/recent/  - custom action (list)
```

<a id="q4"></a>
### Q4: How do you handle authentication in DRF?
**Answer:**

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
        'rest_framework.authentication.SessionAuthentication',
    ],
}

# Token Authentication
# pip install djangorestframework
INSTALLED_APPS = [
    # ...
    'rest_framework.authtoken',
]

# Create tokens for existing users
from rest_framework.authtoken.models import Token
from django.contrib.auth.models import User

for user in User.objects.all():
    Token.objects.get_or_create(user=user)

# Login view to get token
from rest_framework.authtoken.views import ObtainAuthToken
from rest_framework.authtoken.models import Token
from rest_framework.response import Response

class CustomAuthToken(ObtainAuthToken):
    def post(self, request, *args, **kwargs):
        serializer = self.serializer_class(data=request.data)
        serializer.is_valid(raise_exception=True)
        user = serializer.validated_data['user']
        token, created = Token.objects.get_or_create(user=user)
        return Response({
            'token': token.key,
            'user_id': user.pk,
            'email': user.email
        })

# URL
urlpatterns = [
    path('api/auth/login/', CustomAuthToken.as_view()),
]

# Client usage:
# curl -X POST -d "username=user&password=pass" http://localhost/api/auth/login/
# curl -H "Authorization: Token <token>" http://localhost/api/articles/

# JWT Authentication (more common in production)
# pip install djangorestframework-simplejwt

from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
)

urlpatterns = [
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
]

# settings.py
from datetime import timedelta

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
}

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=60),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=1),
    'ROTATE_REFRESH_TOKENS': True,
}

# Custom JWT claims
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer
from rest_framework_simplejwt.views import TokenObtainPairView

class CustomTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        token['username'] = user.username
        token['email'] = user.email
        return token

class CustomTokenObtainPairView(TokenObtainPairView):
    serializer_class = CustomTokenObtainPairSerializer
```

<a id="q5"></a>
### Q5: How do you implement pagination in DRF?
**Answer:**

```python
# settings.py - Global pagination
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
}

# Custom pagination classes
from rest_framework.pagination import (
    PageNumberPagination,
    LimitOffsetPagination,
    CursorPagination
)

# Page number pagination
class StandardPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100
    
    def get_paginated_response(self, data):
        return Response({
            'count': self.page.paginator.count,
            'total_pages': self.page.paginator.num_pages,
            'current_page': self.page.number,
            'next': self.get_next_link(),
            'previous': self.get_previous_link(),
            'results': data
        })

# Usage: /api/articles/?page=2&page_size=10

# Limit/Offset pagination
class LimitOffsetPagination(LimitOffsetPagination):
    default_limit = 20
    max_limit = 100

# Usage: /api/articles/?limit=10&offset=20

# Cursor pagination (best for large datasets)
class ArticleCursorPagination(CursorPagination):
    page_size = 20
    ordering = '-created_at'  # Must be unique, orderable field
    cursor_query_param = 'cursor'

# Usage: /api/articles/?cursor=cD0yMDIwLTA...

# Apply to ViewSet
class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    pagination_class = StandardPagination

# Disable pagination for specific view
class ArticleViewSet(viewsets.ModelViewSet):
    pagination_class = None  # No pagination
    
    @action(detail=False)
    def all(self, request):
        # Custom action without pagination
        self.pagination_class = None
        queryset = self.get_queryset()
        serializer = self.get_serializer(queryset, many=True)
        return Response(serializer.data)
```

---

## Authentication & Authorization

<a id="q6"></a>
### Q6: How does Django's authentication system work?
**Answer:**

```python
# Built-in User model
from django.contrib.auth.models import User

# Create user
user = User.objects.create_user(
    username='john',
    email='john@example.com',
    password='secure_password'
)

# Create superuser
User.objects.create_superuser('admin', 'admin@example.com', 'admin_pass')

# Authentication
from django.contrib.auth import authenticate, login, logout

def login_view(request):
    if request.method == 'POST':
        username = request.POST['username']
        password = request.POST['password']
        
        user = authenticate(request, username=username, password=password)
        
        if user is not None:
            login(request, user)
            return redirect('home')
        else:
            return render(request, 'login.html', {'error': 'Invalid credentials'})
    
    return render(request, 'login.html')

def logout_view(request):
    logout(request)
    return redirect('home')

# Check authentication in views
from django.contrib.auth.decorators import login_required

@login_required
def profile(request):
    return render(request, 'profile.html')

@login_required(login_url='/custom-login/')
def dashboard(request):
    return render(request, 'dashboard.html')

# In templates
"""
{% if user.is_authenticated %}
    <p>Welcome, {{ user.username }}!</p>
    <a href="{% url 'logout' %}">Logout</a>
{% else %}
    <a href="{% url 'login' %}">Login</a>
{% endif %}
"""

# Class-based view authentication
from django.contrib.auth.mixins import LoginRequiredMixin

class ProfileView(LoginRequiredMixin, View):
    login_url = '/login/'
    redirect_field_name = 'next'
    
    def get(self, request):
        return render(request, 'profile.html')

# Built-in auth views
from django.contrib.auth import views as auth_views

urlpatterns = [
    path('login/', auth_views.LoginView.as_view(), name='login'),
    path('logout/', auth_views.LogoutView.as_view(), name='logout'),
    path('password_change/', auth_views.PasswordChangeView.as_view(), name='password_change'),
    path('password_reset/', auth_views.PasswordResetView.as_view(), name='password_reset'),
]
```

<a id="q7"></a>
### Q7: How do you implement custom user models?
**Answer:**

```python
# accounts/models.py
from django.contrib.auth.models import AbstractUser, AbstractBaseUser, BaseUserManager
from django.db import models

# Method 1: Extend AbstractUser (easier)
class CustomUser(AbstractUser):
    email = models.EmailField(unique=True)
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to='avatars/', null=True, blank=True)
    date_of_birth = models.DateField(null=True, blank=True)
    
    # Use email as username
    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['username']
    
    class Meta:
        db_table = 'users'

# Method 2: Extend AbstractBaseUser (full control)
class CustomUserManager(BaseUserManager):
    def create_user(self, email, password=None, **extra_fields):
        if not email:
            raise ValueError('Email is required')
        email = self.normalize_email(email)
        user = self.model(email=email, **extra_fields)
        user.set_password(password)
        user.save(using=self._db)
        return user
    
    def create_superuser(self, email, password=None, **extra_fields):
        extra_fields.setdefault('is_staff', True)
        extra_fields.setdefault('is_superuser', True)
        return self.create_user(email, password, **extra_fields)

class CustomUser(AbstractBaseUser):
    email = models.EmailField(unique=True)
    name = models.CharField(max_length=150)
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)
    is_superuser = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    
    objects = CustomUserManager()
    
    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['name']
    
    def has_perm(self, perm, obj=None):
        return self.is_superuser
    
    def has_module_perms(self, app_label):
        return self.is_superuser

# settings.py
AUTH_USER_MODEL = 'accounts.CustomUser'

# accounts/admin.py
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin
from .models import CustomUser

class CustomUserAdmin(UserAdmin):
    model = CustomUser
    list_display = ['email', 'name', 'is_staff', 'is_active']
    list_filter = ['is_staff', 'is_active']
    fieldsets = (
        (None, {'fields': ('email', 'password')}),
        ('Personal', {'fields': ('name', 'bio', 'avatar')}),
        ('Permissions', {'fields': ('is_staff', 'is_active', 'is_superuser')}),
    )
    add_fieldsets = (
        (None, {
            'classes': ('wide',),
            'fields': ('email', 'name', 'password1', 'password2', 'is_staff', 'is_active')}
        ),
    )
    search_fields = ['email', 'name']
    ordering = ['email']

admin.site.register(CustomUser, CustomUserAdmin)

# Using custom user in other models
from django.conf import settings

class Article(models.Model):
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL,  # Use this instead of User
        on_delete=models.CASCADE
    )
```

<a id="q8"></a>
### Q8: What are permissions and how do you use them?
**Answer:**

```python
# Django's built-in permissions (auto-created for models)
# - add_<model>
# - change_<model>
# - delete_<model>
# - view_<model>

# Check permissions
user.has_perm('myapp.add_article')
user.has_perm('myapp.change_article')

# In views
from django.contrib.auth.decorators import permission_required

@permission_required('myapp.add_article', raise_exception=True)
def create_article(request):
    pass

# Multiple permissions
@permission_required(['myapp.add_article', 'myapp.change_article'])
def manage_article(request):
    pass

# Class-based views
from django.contrib.auth.mixins import PermissionRequiredMixin

class ArticleCreateView(PermissionRequiredMixin, CreateView):
    permission_required = 'myapp.add_article'
    # Or multiple: permission_required = ['myapp.add_article', 'myapp.publish_article']

# Custom permissions in model
class Article(models.Model):
    title = models.CharField(max_length=200)
    
    class Meta:
        permissions = [
            ("publish_article", "Can publish article"),
            ("feature_article", "Can feature article"),
        ]

# DRF permissions
from rest_framework.permissions import (
    IsAuthenticated,
    IsAdminUser,
    AllowAny,
    IsAuthenticatedOrReadOnly,
    BasePermission
)

class ArticleViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated]

# Custom DRF permission
class IsAuthorOrReadOnly(BasePermission):
    def has_permission(self, request, view):
        # Allow any read request
        if request.method in ['GET', 'HEAD', 'OPTIONS']:
            return True
        return request.user.is_authenticated
    
    def has_object_permission(self, request, view, obj):
        # Allow read for any request
        if request.method in ['GET', 'HEAD', 'OPTIONS']:
            return True
        # Write permissions only for author
        return obj.author == request.user

class ArticleViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthorOrReadOnly]
    
    def get_permissions(self):
        if self.action == 'create':
            return [IsAuthenticated()]
        elif self.action in ['update', 'partial_update', 'destroy']:
            return [IsAuthorOrReadOnly()]
        return [AllowAny()]

# Object-level permissions with django-guardian
# pip install django-guardian
from guardian.shortcuts import assign_perm, get_objects_for_user

# Assign permission
assign_perm('change_article', user, article)
assign_perm('delete_article', group, article)

# Check object permission
user.has_perm('change_article', article)

# Get objects user has permission for
articles = get_objects_for_user(user, 'myapp.change_article')
```

---

## Signals & Async

<a id="q9"></a>
### Q9: What are Django signals?
**Answer:**

Signals allow decoupled applications to get notified when certain actions occur.

```python
from django.db.models.signals import pre_save, post_save, pre_delete, post_delete
from django.dispatch import receiver
from django.contrib.auth.signals import user_logged_in, user_logged_out
from .models import Article, Profile

# Using @receiver decorator
@receiver(post_save, sender=Article)
def article_saved(sender, instance, created, **kwargs):
    if created:
        print(f"New article created: {instance.title}")
        # Send notification, update cache, etc.
    else:
        print(f"Article updated: {instance.title}")

# Create profile when user is created
from django.contrib.auth import get_user_model
User = get_user_model()

@receiver(post_save, sender=User)
def create_user_profile(sender, instance, created, **kwargs):
    if created:
        Profile.objects.create(user=instance)

@receiver(post_save, sender=User)
def save_user_profile(sender, instance, **kwargs):
    instance.profile.save()

# Pre-save signal (modify before saving)
@receiver(pre_save, sender=Article)
def article_pre_save(sender, instance, **kwargs):
    if not instance.slug:
        from django.utils.text import slugify
        instance.slug = slugify(instance.title)

# Delete related files when model deleted
@receiver(post_delete, sender=Profile)
def delete_avatar(sender, instance, **kwargs):
    if instance.avatar:
        instance.avatar.delete(save=False)

# Login/logout signals
@receiver(user_logged_in)
def user_logged_in_handler(sender, request, user, **kwargs):
    print(f"User {user.username} logged in")
    # Update last login IP, send notification, etc.

# Manual signal connection (alternative to decorator)
def my_handler(sender, **kwargs):
    pass

post_save.connect(my_handler, sender=Article)

# Custom signals
from django.dispatch import Signal

# Define signal
article_published = Signal()  # No arguments needed in Django 3.0+

# Send signal
class Article(models.Model):
    def publish(self):
        self.status = 'published'
        self.published_at = timezone.now()
        self.save()
        article_published.send(sender=self.__class__, article=self)

# Receive signal
@receiver(article_published)
def notify_subscribers(sender, article, **kwargs):
    # Send emails to subscribers
    pass

# Best practices:
# 1. Put signals in signals.py
# 2. Import signals in apps.py ready() method
# 3. Keep signal handlers simple
# 4. Consider using Celery for heavy tasks

# apps.py
class MyAppConfig(AppConfig):
    name = 'myapp'
    
    def ready(self):
        import myapp.signals  # noqa
```

<a id="q10"></a>
### Q10: How do you use async views in Django?
**Answer:**

```python
# Django 3.1+ supports async views
import asyncio
import httpx
from django.http import JsonResponse

# Async function-based view
async def async_view(request):
    await asyncio.sleep(1)  # Async operation
    return JsonResponse({'message': 'Hello from async!'})

# Async with external API calls
async def fetch_data(request):
    async with httpx.AsyncClient() as client:
        response = await client.get('https://api.example.com/data')
        return JsonResponse(response.json())

# Multiple concurrent requests
async def fetch_multiple(request):
    async with httpx.AsyncClient() as client:
        tasks = [
            client.get('https://api.example.com/users'),
            client.get('https://api.example.com/posts'),
            client.get('https://api.example.com/comments'),
        ]
        responses = await asyncio.gather(*tasks)
        
        return JsonResponse({
            'users': responses[0].json(),
            'posts': responses[1].json(),
            'comments': responses[2].json(),
        })

# Async class-based view
from django.views import View

class AsyncView(View):
    async def get(self, request):
        await asyncio.sleep(1)
        return JsonResponse({'message': 'Async CBV'})

# Async ORM operations (Django 4.1+)
from django.contrib.auth.models import User

async def get_users(request):
    users = [user async for user in User.objects.all()]
    # Or: users = await sync_to_async(list)(User.objects.all())
    return JsonResponse({'count': len(users)})

# Sync to async wrapper
from asgiref.sync import sync_to_async

@sync_to_async
def get_user_sync(user_id):
    return User.objects.get(id=user_id)

async def async_view(request, user_id):
    user = await get_user_sync(user_id)
    return JsonResponse({'username': user.username})

# ASGI configuration (asgi.py)
import os
from django.core.asgi import get_asgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myproject.settings')
application = get_asgi_application()

# Run with ASGI server
# uvicorn myproject.asgi:application
# daphne myproject.asgi:application

# Mixed sync/async
from asgiref.sync import async_to_sync

def sync_view(request):
    # Call async function from sync view
    result = async_to_sync(some_async_function)()
    return JsonResponse(result)
```

---

## Performance & Caching

<a id="q11"></a>
### Q11: How do you optimize Django QuerySets?
**Answer:**

```python
from django.db.models import Prefetch, Count, F, Q

# 1. select_related - ForeignKey/OneToOne (single query with JOIN)
# BAD - N+1 queries
articles = Article.objects.all()
for article in articles:
    print(article.author.username)  # Extra query each time!

# GOOD - Single query with JOIN
articles = Article.objects.select_related('author', 'category')
for article in articles:
    print(article.author.username)  # No extra query

# 2. prefetch_related - ManyToMany/reverse ForeignKey (separate queries)
# BAD - N+1 queries
articles = Article.objects.all()
for article in articles:
    for tag in article.tags.all():  # Extra query each time!
        print(tag.name)

# GOOD - Two queries total
articles = Article.objects.prefetch_related('tags')

# Custom prefetch
articles = Article.objects.prefetch_related(
    Prefetch(
        'comments',
        queryset=Comment.objects.filter(is_approved=True).select_related('author'),
        to_attr='approved_comments'
    )
)

# 3. only() and defer() - Load specific fields
# Only load specific fields
articles = Article.objects.only('title', 'slug')

# Defer specific fields (load all except these)
articles = Article.objects.defer('content')

# 4. values() and values_list() - Return dicts/tuples instead of objects
titles = Article.objects.values_list('title', flat=True)
articles = Article.objects.values('id', 'title', 'author__username')

# 5. Use iterator() for large querysets
for article in Article.objects.iterator(chunk_size=1000):
    process(article)

# 6. exists() instead of count() or bool()
# BAD
if Article.objects.filter(status='published').count() > 0:
    pass
# GOOD
if Article.objects.filter(status='published').exists():
    pass

# 7. update() instead of save() for bulk updates
# BAD
for article in Article.objects.filter(status='draft'):
    article.status = 'published'
    article.save()

# GOOD
Article.objects.filter(status='draft').update(status='published')

# 8. bulk_create() for multiple inserts
articles = [Article(title=f'Article {i}') for i in range(1000)]
Article.objects.bulk_create(articles, batch_size=100)

# 9. Use F() expressions for database-level operations
Article.objects.update(view_count=F('view_count') + 1)

# 10. Use indexes
class Article(models.Model):
    title = models.CharField(max_length=200, db_index=True)
    
    class Meta:
        indexes = [
            models.Index(fields=['status', 'published_at']),
            models.Index(fields=['-created_at']),
        ]

# 11. Raw SQL for complex queries
from django.db import connection

with connection.cursor() as cursor:
    cursor.execute("SELECT * FROM articles WHERE ...")
    rows = cursor.fetchall()

# Or with ORM
Article.objects.raw('SELECT * FROM articles WHERE ...')
```

<a id="q12"></a>
### Q12: How does caching work in Django?
**Answer:**

```python
# settings.py - Cache backends
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
    }
}

# Other backends:
# - django.core.cache.backends.memcached.PyMemcacheCache
# - django.core.cache.backends.db.DatabaseCache
# - django.core.cache.backends.filebased.FileBasedCache
# - django.core.cache.backends.locmem.LocMemCache

# Low-level cache API
from django.core.cache import cache

# Set cache
cache.set('my_key', 'my_value', timeout=300)  # 5 minutes

# Get cache
value = cache.get('my_key', default='default_value')

# Delete cache
cache.delete('my_key')

# Get or set pattern
def get_articles():
    return Article.objects.all()

articles = cache.get_or_set('articles', get_articles, timeout=300)

# Multiple operations
cache.set_many({'key1': 'value1', 'key2': 'value2'})
values = cache.get_many(['key1', 'key2'])
cache.delete_many(['key1', 'key2'])

# Increment/decrement
cache.set('counter', 0)
cache.incr('counter')
cache.decr('counter')

# Per-view caching
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)  # 15 minutes
def article_list(request):
    return render(request, 'articles/list.html')

# With vary headers
from django.views.decorators.vary import vary_on_headers

@vary_on_headers('User-Agent')
@cache_page(60 * 15)
def my_view(request):
    pass

# Template fragment caching
"""
{% load cache %}

{% cache 500 sidebar request.user.username %}
    ... expensive sidebar content ...
{% endcache %}
"""

# Cache in class-based views
from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page

@method_decorator(cache_page(60 * 15), name='dispatch')
class ArticleListView(ListView):
    model = Article

# Conditional caching
from django.views.decorators.http import condition

def latest_article_etag(request):
    return Article.objects.latest('updated_at').updated_at.isoformat()

@condition(etag_func=latest_article_etag)
def article_list(request):
    pass

# Cache invalidation
def save_article(article):
    article.save()
    cache.delete('articles')
    cache.delete(f'article_{article.id}')

# Using signals for cache invalidation
@receiver(post_save, sender=Article)
def invalidate_article_cache(sender, instance, **kwargs):
    cache.delete('articles')
    cache.delete(f'article_{instance.id}')
```

---

## Deployment & Security

<a id="q13"></a>
### Q13: What security features does Django provide?
**Answer:**

```python
# 1. CSRF Protection (enabled by default)
# Template:
"""
<form method="post">
    {% csrf_token %}
    ...
</form>
"""

# Exempt specific view
from django.views.decorators.csrf import csrf_exempt

@csrf_exempt
def my_api_view(request):
    pass

# 2. XSS Protection
# Django auto-escapes template variables
"""
{{ user_input }}  {# Safe - auto-escaped #}
{{ user_input|safe }}  {# UNSAFE - use carefully #}
"""

# 3. SQL Injection Protection
# ORM is safe
Article.objects.filter(title=user_input)  # Safe

# Raw SQL - use parameterized queries
Article.objects.raw('SELECT * FROM articles WHERE title = %s', [user_input])

# 4. Clickjacking Protection
# settings.py
X_FRAME_OPTIONS = 'DENY'  # or 'SAMEORIGIN'

# Or per-view
from django.views.decorators.clickjacking import xframe_options_deny

@xframe_options_deny
def my_view(request):
    pass

# 5. SSL/HTTPS
# settings.py (production)
SECURE_SSL_REDIRECT = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000  # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

# 6. Password Hashing (automatic)
user.set_password('new_password')  # Uses PBKDF2 by default

# settings.py - Password validators
AUTH_PASSWORD_VALIDATORS = [
    {'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator'},
    {'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
     'OPTIONS': {'min_length': 10}},
    {'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator'},
    {'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator'},
]

# 7. Content Security Policy
# pip install django-csp
# settings.py
CSP_DEFAULT_SRC = ("'self'",)
CSP_SCRIPT_SRC = ("'self'", 'cdn.example.com')
CSP_STYLE_SRC = ("'self'", "'unsafe-inline'")

# 8. Host Header Validation
ALLOWED_HOSTS = ['example.com', 'www.example.com']

# 9. Security Middleware
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    # ...
]

# 10. Secrets Management
# Never commit secrets to version control
# Use environment variables
import os
SECRET_KEY = os.environ.get('DJANGO_SECRET_KEY')

# Or django-environ
import environ
env = environ.Env()
SECRET_KEY = env('SECRET_KEY')

# Debug security check
# python manage.py check --deploy
```

<a id="q14"></a>
### Q14: How do you deploy Django applications?
**Answer:**

```python
# Production settings
DEBUG = False
ALLOWED_HOSTS = ['example.com', 'www.example.com']

# Static files
STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'

# Collect static files
# python manage.py collectstatic

# Media files
MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'

# WSGI/ASGI server configuration

# Gunicorn (WSGI)
# pip install gunicorn
# gunicorn myproject.wsgi:application --bind 0.0.0.0:8000 --workers 3

# Uvicorn (ASGI)
# pip install uvicorn
# uvicorn myproject.asgi:application --host 0.0.0.0 --port 8000

# Docker deployment
"""
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN python manage.py collectstatic --noinput

EXPOSE 8000

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "myproject.wsgi:application"]
"""

"""
# docker-compose.yml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DEBUG=False
      - DATABASE_URL=postgres://user:pass@db:5432/dbname
    depends_on:
      - db
      - redis
  
  db:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=dbname
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
  
  redis:
    image: redis:7
  
  celery:
    build: .
    command: celery -A myproject worker -l info
    depends_on:
      - redis

volumes:
  postgres_data:
"""

# Nginx configuration
"""
# /etc/nginx/sites-available/myproject
server {
    listen 80;
    server_name example.com;
    
    location /static/ {
        alias /path/to/staticfiles/;
    }
    
    location /media/ {
        alias /path/to/media/;
    }
    
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
"""

# Systemd service
"""
# /etc/systemd/system/gunicorn.service
[Unit]
Description=Gunicorn Django
After=network.target

[Service]
User=www-data
Group=www-data
WorkingDirectory=/path/to/project
ExecStart=/path/to/venv/bin/gunicorn --workers 3 --bind unix:/run/gunicorn.sock myproject.wsgi:application

[Install]
WantedBy=multi-user.target
"""
```

<a id="q15"></a>
### Q15: What is Celery and how do you use it with Django?
**Answer:**

Celery is a distributed task queue for handling async tasks.

```python
# Installation
# pip install celery redis

# myproject/celery.py
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myproject.settings')

app = Celery('myproject')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()

# myproject/__init__.py
from .celery import app as celery_app
__all__ = ('celery_app',)

# settings.py
CELERY_BROKER_URL = 'redis://localhost:6379/0'
CELERY_RESULT_BACKEND = 'redis://localhost:6379/0'
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = 'UTC'

# Define tasks - myapp/tasks.py
from celery import shared_task
from django.core.mail import send_mail

@shared_task
def send_email_task(subject, message, recipient):
    send_mail(subject, message, 'from@example.com', [recipient])
    return f"Email sent to {recipient}"

@shared_task(bind=True, max_retries=3)
def process_data_task(self, data_id):
    try:
        # Process data
        data = Data.objects.get(id=data_id)
        data.process()
    except Exception as exc:
        self.retry(exc=exc, countdown=60)

# Call tasks
from myapp.tasks import send_email_task

# Async call
send_email_task.delay('Subject', 'Message', 'user@example.com')

# With options
send_email_task.apply_async(
    args=['Subject', 'Message', 'user@example.com'],
    countdown=60,  # Wait 60 seconds
    expires=3600,  # Expire after 1 hour
)

# Get result
result = send_email_task.delay('Subject', 'Message', 'user@example.com')
print(result.id)  # Task ID
print(result.status)  # PENDING, STARTED, SUCCESS, FAILURE
print(result.get(timeout=10))  # Wait for result

# Periodic tasks (celery beat)
# pip install django-celery-beat

# settings.py
INSTALLED_APPS = [
    # ...
    'django_celery_beat',
]

CELERY_BEAT_SCHEDULER = 'django_celery_beat.schedulers:DatabaseScheduler'

# Or in code
from celery.schedules import crontab

app.conf.beat_schedule = {
    'cleanup-every-day': {
        'task': 'myapp.tasks.cleanup_old_data',
        'schedule': crontab(hour=0, minute=0),
    },
    'send-report-every-monday': {
        'task': 'myapp.tasks.send_weekly_report',
        'schedule': crontab(hour=9, minute=0, day_of_week=1),
    },
}

# Run celery
# celery -A myproject worker -l info
# celery -A myproject beat -l info
```

---

[← Django Basics](django-basics.md) | [Back to Python Index](README.md) | [Python Testing →](python-testing.md)
