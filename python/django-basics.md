# Django Basics Interview Questions

## Table of Contents

### Django Architecture
- [Q1: What is Django and what is the MTV architecture?](#q1)
- [Q2: Explain the Django project structure](#q2)
- [Q3: What is the difference between a project and an app in Django?](#q3)
- [Q4: What is the request/response cycle in Django?](#q4)

### Models & ORM
- [Q5: How do you define a model in Django?](#q5)
- [Q6: What are Django migrations and how do they work?](#q6)
- [Q7: Explain Django ORM QuerySets](#q7)
- [Q8: What are model relationships in Django?](#q8)
- [Q9: What is the difference between `filter()`, `get()`, and `exclude()`?](#q9)
- [Q10: How do you perform complex queries with Q objects?](#q10)

### Views
- [Q11: What are function-based views (FBVs)?](#q11)
- [Q12: What are class-based views (CBVs)?](#q12)
- [Q13: What are generic views in Django?](#q13)
- [Q14: How do you handle forms in Django views?](#q14)

### URLs & Templates
- [Q15: How does URL routing work in Django?](#q15)
- [Q16: What is the Django template language?](#q16)
- [Q17: What are template tags and filters?](#q17)
- [Q18: How do you use template inheritance?](#q18)

### Configuration & Middleware
- [Q19: What is Django middleware?](#q19)
- [Q20: How do you configure Django settings for different environments?](#q20)

---

## Django Architecture

<a id="q1"></a>
### Q1: What is Django and what is the MTV architecture?
**Answer:**

Django is a high-level Python web framework that follows the **MTV (Model-Template-View)** pattern, which is Django's interpretation of MVC.

| Component | MTV (Django) | MVC Equivalent | Purpose |
|-----------|--------------|----------------|---------|
| Model | Model | Model | Data structure, database interaction |
| Template | Template | View | Presentation layer, HTML rendering |
| View | View | Controller | Business logic, request handling |

```python
# Model - defines data structure
from django.db import models

class Article(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    
    def __str__(self):
        return self.title

# View - handles business logic
from django.shortcuts import render
from .models import Article

def article_list(request):
    articles = Article.objects.all()
    return render(request, 'articles/list.html', {'articles': articles})

# Template (articles/list.html) - presentation
"""
{% for article in articles %}
    <h2>{{ article.title }}</h2>
    <p>{{ article.content }}</p>
{% endfor %}
"""

# URL configuration - maps URLs to views
from django.urls import path
from . import views

urlpatterns = [
    path('articles/', views.article_list, name='article-list'),
]
```

**Django's key features:**
- ORM (Object-Relational Mapping)
- Admin interface (auto-generated)
- URL routing
- Template engine
- Form handling
- Authentication system
- Security features (CSRF, XSS protection)

<a id="q2"></a>
### Q2: Explain the Django project structure
**Answer:**

```
myproject/                  # Project root
├── manage.py              # Command-line utility
├── myproject/             # Project configuration package
│   ├── __init__.py       # Python package marker
│   ├── settings.py       # Project settings
│   ├── urls.py           # Root URL configuration
│   ├── asgi.py           # ASGI config (async)
│   └── wsgi.py           # WSGI config (deployment)
├── myapp/                 # Application directory
│   ├── __init__.py
│   ├── admin.py          # Admin site configuration
│   ├── apps.py           # App configuration
│   ├── models.py         # Database models
│   ├── views.py          # View functions/classes
│   ├── urls.py           # App URL patterns
│   ├── forms.py          # Form definitions
│   ├── serializers.py    # DRF serializers (if using DRF)
│   ├── tests.py          # Unit tests
│   ├── migrations/       # Database migrations
│   │   └── __init__.py
│   ├── templates/        # App-specific templates
│   │   └── myapp/
│   │       └── template.html
│   └── static/           # App-specific static files
│       └── myapp/
│           ├── css/
│           └── js/
├── templates/             # Project-wide templates
├── static/               # Project-wide static files
├── media/                # User-uploaded files
├── requirements.txt      # Python dependencies
└── .env                  # Environment variables
```

```python
# settings.py - Key configurations
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'myapp',  # Your app
]

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydatabase',
        'USER': 'myuser',
        'PASSWORD': 'mypassword',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],  # Project templates
        'APP_DIRS': True,  # Look in app/templates/
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
```

<a id="q3"></a>
### Q3: What is the difference between a project and an app in Django?
**Answer:**

| Aspect | Project | App |
|--------|---------|-----|
| Definition | Collection of configurations and apps | Reusable component with specific functionality |
| Scope | Entire website | Single feature |
| Contains | settings.py, root urls.py | models.py, views.py, templates |
| Reusability | Not reusable | Can be reused across projects |
| Example | E-commerce website | User authentication, Product catalog, Shopping cart |

```bash
# Create project
django-admin startproject myproject

# Create apps within project
cd myproject
python manage.py startapp users
python manage.py startapp products
python manage.py startapp orders
```

```python
# Register apps in settings.py
INSTALLED_APPS = [
    # Django built-in apps
    'django.contrib.admin',
    'django.contrib.auth',
    # ...
    
    # Third-party apps
    'rest_framework',
    'corsheaders',
    
    # Your apps
    'users.apps.UsersConfig',
    'products.apps.ProductsConfig',
    'orders.apps.OrdersConfig',
]

# Each app should be self-contained
# products/
#   models.py     - Product, Category models
#   views.py      - Product list, detail views
#   urls.py       - /products/, /products/<id>/
#   admin.py      - ProductAdmin
#   templates/products/
#   tests.py

# Include app URLs in project's urls.py
# myproject/urls.py
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('users/', include('users.urls')),
    path('products/', include('products.urls')),
    path('orders/', include('orders.urls')),
]
```

<a id="q4"></a>
### Q4: What is the request/response cycle in Django?
**Answer:**

```
Browser Request
      │
      ▼
┌─────────────────┐
│   WSGI Server   │  (Gunicorn, uWSGI)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Middleware    │  (Request phase - top to bottom)
│   - Security    │
│   - Session     │
│   - Auth        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  URL Resolver   │  (Match URL to view)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│      View       │  (Process request, call models)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│    Template     │  (Render HTML)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Middleware    │  (Response phase - bottom to top)
└────────┬────────┘
         │
         ▼
   Browser Response
```

```python
# The request object
from django.http import HttpRequest

def my_view(request: HttpRequest):
    # Request attributes
    request.method      # 'GET', 'POST', etc.
    request.GET         # QueryDict of GET parameters
    request.POST        # QueryDict of POST parameters
    request.COOKIES     # Dict of cookies
    request.FILES       # Uploaded files
    request.user        # Current user (from auth middleware)
    request.session     # Session dict
    request.path        # URL path
    request.META        # HTTP headers as dict
    
    # Common patterns
    if request.method == 'POST':
        # Handle form submission
        pass
    
    # Access headers
    auth_header = request.META.get('HTTP_AUTHORIZATION')
    
    # Check authentication
    if request.user.is_authenticated:
        pass

# The response object
from django.http import HttpResponse, JsonResponse
from django.shortcuts import render, redirect

def my_view(request):
    # Simple text response
    return HttpResponse("Hello, World!")
    
    # HTML template response
    return render(request, 'template.html', {'context': 'data'})
    
    # JSON response
    return JsonResponse({'key': 'value'})
    
    # Redirect response
    return redirect('view-name')
    
    # Custom status code
    return HttpResponse("Not Found", status=404)

# Response attributes
response = HttpResponse("Hello")
response.status_code = 200
response['Content-Type'] = 'text/html'
response.set_cookie('key', 'value')
```

---

## Models & ORM

<a id="q5"></a>
### Q5: How do you define a model in Django?
**Answer:**

```python
from django.db import models
from django.contrib.auth.models import User

class Category(models.Model):
    name = models.CharField(max_length=100)
    slug = models.SlugField(unique=True)
    
    class Meta:
        verbose_name_plural = "categories"
        ordering = ['name']
    
    def __str__(self):
        return self.name

class Article(models.Model):
    # Choices for status field
    class Status(models.TextChoices):
        DRAFT = 'draft', 'Draft'
        PUBLISHED = 'published', 'Published'
    
    # Field types
    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    content = models.TextField()
    excerpt = models.TextField(blank=True)  # Optional
    
    # Relationships
    author = models.ForeignKey(User, on_delete=models.CASCADE, related_name='articles')
    category = models.ForeignKey(Category, on_delete=models.SET_NULL, null=True)
    tags = models.ManyToManyField('Tag', blank=True)
    
    # Status and dates
    status = models.CharField(max_length=10, choices=Status.choices, default=Status.DRAFT)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    published_at = models.DateTimeField(null=True, blank=True)
    
    # Boolean and numeric
    is_featured = models.BooleanField(default=False)
    view_count = models.PositiveIntegerField(default=0)
    
    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['slug']),
            models.Index(fields=['status', 'published_at']),
        ]
        constraints = [
            models.UniqueConstraint(fields=['author', 'slug'], name='unique_author_slug')
        ]
    
    def __str__(self):
        return self.title
    
    def get_absolute_url(self):
        from django.urls import reverse
        return reverse('article-detail', kwargs={'slug': self.slug})
    
    @property
    def is_published(self):
        return self.status == self.Status.PUBLISHED
    
    def publish(self):
        from django.utils import timezone
        self.status = self.Status.PUBLISHED
        self.published_at = timezone.now()
        self.save()

class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True)
    
    def __str__(self):
        return self.name
```

**Common field types:**

| Field Type | Use Case |
|------------|----------|
| CharField | Short text (max_length required) |
| TextField | Long text |
| IntegerField | Integers |
| FloatField | Floating point |
| DecimalField | Precise decimals (money) |
| BooleanField | True/False |
| DateField | Date only |
| DateTimeField | Date and time |
| EmailField | Email validation |
| URLField | URL validation |
| FileField | File uploads |
| ImageField | Image uploads |
| ForeignKey | Many-to-one |
| ManyToManyField | Many-to-many |
| OneToOneField | One-to-one |

<a id="q6"></a>
### Q6: What are Django migrations and how do they work?
**Answer:**

Migrations are Django's way of propagating model changes to the database schema.

```bash
# Create migrations from model changes
python manage.py makemigrations

# Create migrations for specific app
python manage.py makemigrations myapp

# Apply migrations
python manage.py migrate

# Show migration status
python manage.py showmigrations

# Show SQL for a migration
python manage.py sqlmigrate myapp 0001

# Rollback migration
python manage.py migrate myapp 0001  # Go back to 0001

# Rollback all migrations for an app
python manage.py migrate myapp zero
```

```python
# Migration file example: myapp/migrations/0001_initial.py
from django.db import migrations, models

class Migration(migrations.Migration):
    initial = True
    
    dependencies = [
        ('auth', '0012_alter_user_first_name_max_length'),
    ]
    
    operations = [
        migrations.CreateModel(
            name='Article',
            fields=[
                ('id', models.BigAutoField(auto_created=True, primary_key=True)),
                ('title', models.CharField(max_length=200)),
                ('content', models.TextField()),
            ],
        ),
    ]

# Custom data migration
from django.db import migrations

def populate_slugs(apps, schema_editor):
    Article = apps.get_model('myapp', 'Article')
    for article in Article.objects.all():
        article.slug = slugify(article.title)
        article.save()

def reverse_populate_slugs(apps, schema_editor):
    pass  # Nothing to do on reverse

class Migration(migrations.Migration):
    dependencies = [
        ('myapp', '0001_initial'),
    ]
    
    operations = [
        migrations.AddField(
            model_name='article',
            name='slug',
            field=models.SlugField(default=''),
        ),
        migrations.RunPython(populate_slugs, reverse_populate_slugs),
        migrations.AlterField(
            model_name='article',
            name='slug',
            field=models.SlugField(unique=True),
        ),
    ]
```

<a id="q7"></a>
### Q7: Explain Django ORM QuerySets
**Answer:**

QuerySets are lazy collections of database objects that allow chaining filters and operations.

```python
from myapp.models import Article
from django.db.models import Q, F, Count, Avg

# Basic queries
articles = Article.objects.all()           # All articles
article = Article.objects.get(id=1)        # Single article (raises if not found)
articles = Article.objects.filter(status='published')
articles = Article.objects.exclude(status='draft')

# QuerySets are LAZY - not evaluated until needed
articles = Article.objects.filter(status='published')  # No DB query yet
print(articles)  # DB query executed here

# Chaining filters
articles = Article.objects.filter(status='published').filter(author_id=1)
# Equivalent to:
articles = Article.objects.filter(status='published', author_id=1)

# Lookups (field__lookup)
Article.objects.filter(title__contains='django')      # Case-sensitive
Article.objects.filter(title__icontains='django')     # Case-insensitive
Article.objects.filter(title__startswith='How')
Article.objects.filter(title__endswith='?')
Article.objects.filter(view_count__gt=100)            # Greater than
Article.objects.filter(view_count__gte=100)           # Greater than or equal
Article.objects.filter(view_count__lt=100)            # Less than
Article.objects.filter(view_count__range=(10, 100))   # Between
Article.objects.filter(published_at__year=2024)       # Date parts
Article.objects.filter(category__isnull=True)         # NULL check
Article.objects.filter(status__in=['draft', 'pending'])

# Related field lookups (spanning relationships)
Article.objects.filter(author__username='john')
Article.objects.filter(category__name='Technology')

# Ordering
Article.objects.order_by('title')          # Ascending
Article.objects.order_by('-created_at')    # Descending
Article.objects.order_by('author__name', '-created_at')

# Limiting
Article.objects.all()[:5]                  # First 5 (LIMIT 5)
Article.objects.all()[5:10]                # OFFSET 5 LIMIT 5

# Values and annotations
Article.objects.values('title', 'author__username')  # Returns dicts
Article.objects.values_list('title', flat=True)      # Returns tuples/values

# Aggregation
from django.db.models import Count, Avg, Sum, Max, Min
Article.objects.count()
Article.objects.aggregate(avg_views=Avg('view_count'))
Article.objects.aggregate(total_views=Sum('view_count'))

# Annotation (per-object aggregation)
authors = User.objects.annotate(article_count=Count('articles'))
for author in authors:
    print(f"{author.username}: {author.article_count} articles")

# F expressions (reference other fields)
Article.objects.filter(updated_at__gt=F('created_at'))
Article.objects.update(view_count=F('view_count') + 1)

# Exists
Article.objects.filter(status='published').exists()

# Select related (eager loading for ForeignKey)
articles = Article.objects.select_related('author', 'category')
# Single query with JOIN

# Prefetch related (eager loading for ManyToMany/reverse FK)
articles = Article.objects.prefetch_related('tags')
# Two queries: one for articles, one for tags
```

<a id="q8"></a>
### Q8: What are model relationships in Django?
**Answer:**

| Relationship | Django Field | Database | Example |
|--------------|--------------|----------|---------|
| One-to-Many | ForeignKey | Foreign key column | Article → Author |
| Many-to-Many | ManyToManyField | Junction table | Article ↔ Tags |
| One-to-One | OneToOneField | Foreign key + unique | User ↔ Profile |

```python
from django.db import models
from django.contrib.auth.models import User

# ONE-TO-MANY (ForeignKey)
class Author(models.Model):
    name = models.CharField(max_length=100)

class Article(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(
        Author,
        on_delete=models.CASCADE,  # Delete articles when author deleted
        related_name='articles'     # Author.articles.all()
    )

# on_delete options:
# CASCADE - Delete related objects
# PROTECT - Prevent deletion
# SET_NULL - Set to NULL (requires null=True)
# SET_DEFAULT - Set to default value
# SET() - Set to specific value
# DO_NOTHING - Do nothing (may cause integrity error)

# Usage
author = Author.objects.get(id=1)
author.articles.all()  # All articles by this author

article = Article.objects.get(id=1)
article.author  # The author object

# MANY-TO-MANY
class Tag(models.Model):
    name = models.CharField(max_length=50)

class Article(models.Model):
    title = models.CharField(max_length=200)
    tags = models.ManyToManyField(Tag, related_name='articles', blank=True)

# Usage
article = Article.objects.get(id=1)
article.tags.all()              # All tags for this article
article.tags.add(tag1, tag2)    # Add tags
article.tags.remove(tag1)       # Remove tag
article.tags.clear()            # Remove all tags
article.tags.set([tag1, tag2])  # Replace all tags

tag = Tag.objects.get(id=1)
tag.articles.all()              # All articles with this tag

# Many-to-Many with through model (extra fields)
class Membership(models.Model):
    person = models.ForeignKey('Person', on_delete=models.CASCADE)
    group = models.ForeignKey('Group', on_delete=models.CASCADE)
    date_joined = models.DateField()
    role = models.CharField(max_length=50)

class Person(models.Model):
    name = models.CharField(max_length=100)
    groups = models.ManyToManyField('Group', through='Membership')

class Group(models.Model):
    name = models.CharField(max_length=100)

# Usage with through model
Membership.objects.create(person=person, group=group, date_joined=date.today(), role='member')

# ONE-TO-ONE
class Profile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name='profile')
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to='avatars/', null=True)

# Usage
user = User.objects.get(id=1)
user.profile.bio  # Access profile

profile = Profile.objects.get(id=1)
profile.user      # Access user

# Create profile when user is created (using signals)
from django.db.models.signals import post_save
from django.dispatch import receiver

@receiver(post_save, sender=User)
def create_profile(sender, instance, created, **kwargs):
    if created:
        Profile.objects.create(user=instance)
```

<a id="q9"></a>
### Q9: What is the difference between `filter()`, `get()`, and `exclude()`?
**Answer:**

| Method | Returns | If not found | Multiple results |
|--------|---------|--------------|------------------|
| `filter()` | QuerySet | Empty QuerySet | All matching |
| `get()` | Single object | DoesNotExist | MultipleObjectsReturned |
| `exclude()` | QuerySet | Empty QuerySet | All non-matching |

```python
from myapp.models import Article
from django.core.exceptions import ObjectDoesNotExist

# filter() - returns QuerySet (possibly empty)
articles = Article.objects.filter(status='published')
print(articles.count())  # 0 if none found
print(articles.exists()) # False if none found

# get() - returns single object
try:
    article = Article.objects.get(id=1)
except Article.DoesNotExist:
    print("Article not found")
except Article.MultipleObjectsReturned:
    print("Multiple articles found")

# Safe get with get_or_404 (in views)
from django.shortcuts import get_object_or_404
article = get_object_or_404(Article, id=1)  # Returns 404 if not found

# get_or_create - get or create if doesn't exist
article, created = Article.objects.get_or_create(
    title='My Article',
    defaults={'content': 'Default content', 'author_id': 1}
)
print(created)  # True if created, False if existed

# update_or_create
article, created = Article.objects.update_or_create(
    slug='my-article',
    defaults={'title': 'Updated Title', 'content': 'Updated content'}
)

# exclude() - returns QuerySet without matching items
articles = Article.objects.exclude(status='draft')

# Combining filter and exclude
articles = Article.objects.filter(author_id=1).exclude(status='draft')

# first() and last() - return single object or None
article = Article.objects.filter(author_id=1).first()
article = Article.objects.order_by('-created_at').first()

# exists() - check if any results
if Article.objects.filter(status='published').exists():
    print("Published articles exist")
```

<a id="q10"></a>
### Q10: How do you perform complex queries with Q objects?
**Answer:**

Q objects allow complex queries with OR, AND, NOT operations.

```python
from django.db.models import Q
from myapp.models import Article

# OR queries (can't do with regular filter)
articles = Article.objects.filter(
    Q(status='published') | Q(author_id=1)
)
# SQL: WHERE status = 'published' OR author_id = 1

# AND queries (multiple ways)
articles = Article.objects.filter(
    Q(status='published') & Q(author_id=1)
)
# Same as:
articles = Article.objects.filter(status='published', author_id=1)

# NOT queries
articles = Article.objects.filter(~Q(status='draft'))
# SQL: WHERE NOT status = 'draft'

# Complex combinations
articles = Article.objects.filter(
    Q(status='published') | Q(status='pending'),
    Q(author_id=1) | Q(author_id=2),
    ~Q(is_featured=True)
)
# SQL: WHERE (status = 'published' OR status = 'pending')
#        AND (author_id = 1 OR author_id = 2)
#        AND NOT is_featured = True

# Dynamic query building
conditions = Q()
if search_term:
    conditions &= Q(title__icontains=search_term) | Q(content__icontains=search_term)
if author_id:
    conditions &= Q(author_id=author_id)
if status:
    conditions &= Q(status=status)

articles = Article.objects.filter(conditions)

# Search across multiple fields
def search_articles(query):
    return Article.objects.filter(
        Q(title__icontains=query) |
        Q(content__icontains=query) |
        Q(author__username__icontains=query) |
        Q(tags__name__icontains=query)
    ).distinct()

# Conditional expressions
from django.db.models import Case, When, Value, CharField

articles = Article.objects.annotate(
    priority=Case(
        When(is_featured=True, then=Value('high')),
        When(view_count__gt=1000, then=Value('medium')),
        default=Value('low'),
        output_field=CharField(),
    )
)
```

---

## Views

<a id="q11"></a>
### Q11: What are function-based views (FBVs)?
**Answer:**

Function-based views are simple Python functions that take a request and return a response.

```python
from django.shortcuts import render, redirect, get_object_or_404
from django.http import HttpResponse, JsonResponse, Http404
from django.views.decorators.http import require_http_methods, require_GET, require_POST
from django.contrib.auth.decorators import login_required
from .models import Article
from .forms import ArticleForm

# Basic view
def article_list(request):
    articles = Article.objects.filter(status='published')
    return render(request, 'articles/list.html', {'articles': articles})

# View with URL parameter
def article_detail(request, slug):
    article = get_object_or_404(Article, slug=slug, status='published')
    return render(request, 'articles/detail.html', {'article': article})

# Handling different HTTP methods
@require_http_methods(["GET", "POST"])
def article_create(request):
    if request.method == 'POST':
        form = ArticleForm(request.POST)
        if form.is_valid():
            article = form.save(commit=False)
            article.author = request.user
            article.save()
            return redirect('article-detail', slug=article.slug)
    else:
        form = ArticleForm()
    
    return render(request, 'articles/form.html', {'form': form})

# Login required
@login_required
def dashboard(request):
    user_articles = Article.objects.filter(author=request.user)
    return render(request, 'articles/dashboard.html', {'articles': user_articles})

# JSON response (for APIs)
def api_articles(request):
    articles = Article.objects.filter(status='published').values('id', 'title', 'created_at')
    return JsonResponse(list(articles), safe=False)

# Handling query parameters
def search(request):
    query = request.GET.get('q', '')
    page = request.GET.get('page', 1)
    
    articles = Article.objects.filter(title__icontains=query) if query else Article.objects.none()
    
    return render(request, 'articles/search.html', {
        'articles': articles,
        'query': query
    })

# Custom decorators
from functools import wraps

def author_required(view_func):
    @wraps(view_func)
    def wrapper(request, *args, **kwargs):
        article = get_object_or_404(Article, slug=kwargs.get('slug'))
        if article.author != request.user:
            return HttpResponse("Forbidden", status=403)
        return view_func(request, *args, **kwargs)
    return wrapper

@login_required
@author_required
def article_edit(request, slug):
    article = get_object_or_404(Article, slug=slug)
    # ... edit logic
```

<a id="q12"></a>
### Q12: What are class-based views (CBVs)?
**Answer:**

Class-based views organize view logic using classes and inheritance.

```python
from django.views import View
from django.views.generic import TemplateView
from django.shortcuts import render, redirect, get_object_or_404
from django.http import HttpResponse
from django.contrib.auth.mixins import LoginRequiredMixin
from .models import Article
from .forms import ArticleForm

# Basic class-based view
class ArticleListView(View):
    def get(self, request):
        articles = Article.objects.filter(status='published')
        return render(request, 'articles/list.html', {'articles': articles})
    
    def post(self, request):
        # Handle POST request
        return HttpResponse("Method not allowed", status=405)

# TemplateView - simple template rendering
class HomeView(TemplateView):
    template_name = 'home.html'
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['featured_articles'] = Article.objects.filter(is_featured=True)[:5]
        return context

# View with mixins
class DashboardView(LoginRequiredMixin, View):
    login_url = '/login/'
    
    def get(self, request):
        articles = Article.objects.filter(author=request.user)
        return render(request, 'dashboard.html', {'articles': articles})

# Custom mixin
class AuthorRequiredMixin:
    def dispatch(self, request, *args, **kwargs):
        article = self.get_object()
        if article.author != request.user:
            return HttpResponse("Forbidden", status=403)
        return super().dispatch(request, *args, **kwargs)

# Complete CRUD example
class ArticleCreateView(LoginRequiredMixin, View):
    template_name = 'articles/form.html'
    
    def get(self, request):
        form = ArticleForm()
        return render(request, self.template_name, {'form': form})
    
    def post(self, request):
        form = ArticleForm(request.POST)
        if form.is_valid():
            article = form.save(commit=False)
            article.author = request.user
            article.save()
            return redirect('article-detail', slug=article.slug)
        return render(request, self.template_name, {'form': form})

# URL configuration
from django.urls import path
from .views import ArticleListView, ArticleCreateView

urlpatterns = [
    path('articles/', ArticleListView.as_view(), name='article-list'),
    path('articles/create/', ArticleCreateView.as_view(), name='article-create'),
]
```

<a id="q13"></a>
### Q13: What are generic views in Django?
**Answer:**

Generic views provide pre-built patterns for common operations.

```python
from django.views.generic import (
    ListView, DetailView, CreateView, UpdateView, DeleteView,
    FormView, RedirectView
)
from django.urls import reverse_lazy
from django.contrib.auth.mixins import LoginRequiredMixin
from .models import Article
from .forms import ArticleForm, ContactForm

# ListView - display list of objects
class ArticleListView(ListView):
    model = Article
    template_name = 'articles/list.html'  # Default: article_list.html
    context_object_name = 'articles'       # Default: object_list
    paginate_by = 10
    ordering = ['-created_at']
    
    def get_queryset(self):
        queryset = super().get_queryset()
        return queryset.filter(status='published')
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['categories'] = Category.objects.all()
        return context

# DetailView - display single object
class ArticleDetailView(DetailView):
    model = Article
    template_name = 'articles/detail.html'
    context_object_name = 'article'
    slug_field = 'slug'
    slug_url_kwarg = 'slug'
    
    def get_queryset(self):
        return Article.objects.filter(status='published')

# CreateView - form to create object
class ArticleCreateView(LoginRequiredMixin, CreateView):
    model = Article
    form_class = ArticleForm
    template_name = 'articles/form.html'
    success_url = reverse_lazy('article-list')
    
    def form_valid(self, form):
        form.instance.author = self.request.user
        return super().form_valid(form)

# UpdateView - form to update object
class ArticleUpdateView(LoginRequiredMixin, UpdateView):
    model = Article
    form_class = ArticleForm
    template_name = 'articles/form.html'
    
    def get_success_url(self):
        return reverse_lazy('article-detail', kwargs={'slug': self.object.slug})
    
    def get_queryset(self):
        return Article.objects.filter(author=self.request.user)

# DeleteView - confirm deletion
class ArticleDeleteView(LoginRequiredMixin, DeleteView):
    model = Article
    template_name = 'articles/confirm_delete.html'
    success_url = reverse_lazy('article-list')
    
    def get_queryset(self):
        return Article.objects.filter(author=self.request.user)

# FormView - generic form handling
class ContactView(FormView):
    template_name = 'contact.html'
    form_class = ContactForm
    success_url = reverse_lazy('contact-success')
    
    def form_valid(self, form):
        form.send_email()
        return super().form_valid(form)

# URL patterns
urlpatterns = [
    path('', ArticleListView.as_view(), name='article-list'),
    path('<slug:slug>/', ArticleDetailView.as_view(), name='article-detail'),
    path('create/', ArticleCreateView.as_view(), name='article-create'),
    path('<slug:slug>/edit/', ArticleUpdateView.as_view(), name='article-update'),
    path('<slug:slug>/delete/', ArticleDeleteView.as_view(), name='article-delete'),
]
```

<a id="q14"></a>
### Q14: How do you handle forms in Django views?
**Answer:**

```python
from django import forms
from django.shortcuts import render, redirect
from .models import Article

# Model form
class ArticleForm(forms.ModelForm):
    class Meta:
        model = Article
        fields = ['title', 'content', 'category', 'tags', 'status']
        widgets = {
            'content': forms.Textarea(attrs={'rows': 10}),
            'tags': forms.CheckboxSelectMultiple(),
        }
    
    def clean_title(self):
        title = self.cleaned_data.get('title')
        if len(title) < 10:
            raise forms.ValidationError("Title must be at least 10 characters")
        return title

# Regular form
class ContactForm(forms.Form):
    name = forms.CharField(max_length=100)
    email = forms.EmailField()
    message = forms.CharField(widget=forms.Textarea)
    
    def send_email(self):
        # Send email logic
        pass

# Function-based view with form
def article_create(request):
    if request.method == 'POST':
        form = ArticleForm(request.POST, request.FILES)
        if form.is_valid():
            article = form.save(commit=False)
            article.author = request.user
            article.save()
            form.save_m2m()  # Save many-to-many relationships
            return redirect('article-detail', slug=article.slug)
    else:
        form = ArticleForm()
    
    return render(request, 'articles/form.html', {'form': form})

# Template for form
"""
<form method="post" enctype="multipart/form-data">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">Submit</button>
</form>

<!-- Or manual rendering -->
<form method="post">
    {% csrf_token %}
    
    <div class="field">
        <label for="{{ form.title.id_for_label }}">Title</label>
        {{ form.title }}
        {% if form.title.errors %}
            <ul class="errors">
            {% for error in form.title.errors %}
                <li>{{ error }}</li>
            {% endfor %}
            </ul>
        {% endif %}
    </div>
    
    <button type="submit">Submit</button>
</form>
"""

# Form with initial data
def article_edit(request, slug):
    article = get_object_or_404(Article, slug=slug)
    
    if request.method == 'POST':
        form = ArticleForm(request.POST, instance=article)
        if form.is_valid():
            form.save()
            return redirect('article-detail', slug=article.slug)
    else:
        form = ArticleForm(instance=article)
    
    return render(request, 'articles/form.html', {'form': form, 'article': article})

# Formset for multiple forms
from django.forms import formset_factory, modelformset_factory

ArticleFormSet = modelformset_factory(Article, form=ArticleForm, extra=3)

def bulk_create(request):
    if request.method == 'POST':
        formset = ArticleFormSet(request.POST)
        if formset.is_valid():
            formset.save()
            return redirect('article-list')
    else:
        formset = ArticleFormSet(queryset=Article.objects.none())
    
    return render(request, 'articles/bulk_form.html', {'formset': formset})
```

---

## URLs & Templates

<a id="q15"></a>
### Q15: How does URL routing work in Django?
**Answer:**

```python
# myproject/urls.py (root URL configuration)
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('pages.urls')),
    path('articles/', include('articles.urls', namespace='articles')),
    path('api/', include('api.urls', namespace='api')),
]

# articles/urls.py (app URL configuration)
from django.urls import path, re_path
from . import views

app_name = 'articles'  # Namespace for URL reversing

urlpatterns = [
    # Basic path
    path('', views.article_list, name='list'),
    
    # Path with parameters
    path('<int:pk>/', views.article_detail, name='detail'),
    path('<slug:slug>/', views.article_detail_by_slug, name='detail-slug'),
    
    # Path converters: str, int, slug, uuid, path
    path('category/<slug:category_slug>/', views.category_articles, name='category'),
    path('archive/<int:year>/<int:month>/', views.archive, name='archive'),
    
    # Multiple parameters
    path('<slug:slug>/comment/<int:comment_id>/', views.comment_detail, name='comment'),
    
    # Custom path converter
    path('<yyyy:year>/', views.year_archive, name='year-archive'),
    
    # Regex patterns (for complex patterns)
    re_path(r'^articles/(?P<year>[0-9]{4})/$', views.year_archive),
]

# Custom path converter
class FourDigitYearConverter:
    regex = '[0-9]{4}'
    
    def to_python(self, value):
        return int(value)
    
    def to_url(self, value):
        return '%04d' % value

# Register converter
from django.urls import register_converter
register_converter(FourDigitYearConverter, 'yyyy')

# URL reversing in Python
from django.urls import reverse
from django.shortcuts import redirect

def my_view(request):
    # Basic reverse
    url = reverse('articles:list')  # '/articles/'
    
    # With arguments
    url = reverse('articles:detail', kwargs={'pk': 1})  # '/articles/1/'
    url = reverse('articles:detail', args=[1])          # Same result
    
    # Redirect shortcut
    return redirect('articles:detail', slug='my-article')

# URL reversing in templates
"""
<a href="{% url 'articles:list' %}">All Articles</a>
<a href="{% url 'articles:detail' pk=article.pk %}">{{ article.title }}</a>
<a href="{% url 'articles:archive' year=2024 month=1 %}">January 2024</a>

<!-- With variable -->
{% url 'articles:detail' slug=article.slug as article_url %}
<a href="{{ article_url }}">{{ article.title }}</a>
"""
```

<a id="q16"></a>
### Q16: What is the Django template language?
**Answer:**

Django's template language provides a way to generate HTML dynamically.

```django
{# This is a comment #}

{% comment %}
Multi-line comment
{% endcomment %}

{# Variables #}
{{ variable }}
{{ article.title }}
{{ user.profile.bio }}
{{ items.0 }}  {# First item #}

{# Filters #}
{{ name|lower }}
{{ name|upper }}
{{ text|truncatewords:30 }}
{{ date|date:"F j, Y" }}
{{ price|floatformat:2 }}
{{ content|safe }}  {# Disable auto-escaping #}
{{ content|escape }}
{{ list|join:", " }}
{{ value|default:"Nothing" }}
{{ text|linebreaks }}
{{ text|striptags }}
{{ items|length }}
{{ text|slugify }}

{# Chaining filters #}
{{ name|lower|capfirst }}

{# Conditional statements #}
{% if user.is_authenticated %}
    <p>Welcome, {{ user.username }}!</p>
{% elif user.is_anonymous %}
    <p>Please log in.</p>
{% else %}
    <p>Something else</p>
{% endif %}

{# Comparison operators #}
{% if count > 10 %}...{% endif %}
{% if status == "published" %}...{% endif %}
{% if item in list %}...{% endif %}
{% if not condition %}...{% endif %}
{% if a and b %}...{% endif %}
{% if a or b %}...{% endif %}

{# For loops #}
{% for article in articles %}
    <h2>{{ article.title }}</h2>
    <p>{{ forloop.counter }}. {{ article.title }}</p>  {# 1-indexed #}
    <p>{{ forloop.counter0 }}. {{ article.title }}</p> {# 0-indexed #}
    {% if forloop.first %}First item!{% endif %}
    {% if forloop.last %}Last item!{% endif %}
{% empty %}
    <p>No articles found.</p>
{% endfor %}

{# Dictionary iteration #}
{% for key, value in dictionary.items %}
    <p>{{ key }}: {{ value }}</p>
{% endfor %}

{# With statement (alias) #}
{% with total=business.employees.count %}
    {{ total }} employee{{ total|pluralize }}
{% endwith %}

{# Include other templates #}
{% include "partials/header.html" %}
{% include "partials/card.html" with article=featured_article %}

{# Static files #}
{% load static %}
<link rel="stylesheet" href="{% static 'css/style.css' %}">
<img src="{% static 'images/logo.png' %}" alt="Logo">

{# URLs #}
<a href="{% url 'article-detail' slug=article.slug %}">Read more</a>
```

<a id="q17"></a>
### Q17: What are template tags and filters?
**Answer:**

**Built-in tags and filters:**

```django
{# TAGS #}

{# autoescape - control HTML escaping #}
{% autoescape off %}
    {{ content }}  {# Won't be escaped #}
{% endautoescape %}

{# block/extends - template inheritance #}
{% extends "base.html" %}
{% block content %}...{% endblock %}

{# csrf_token - CSRF protection #}
<form method="post">
    {% csrf_token %}
</form>

{# cycle - alternate between values #}
{% for item in items %}
    <tr class="{% cycle 'odd' 'even' %}">
{% endfor %}

{# firstof - first truthy value #}
{% firstof var1 var2 var3 "default" %}

{# now - current date/time #}
{% now "Y-m-d H:i" %}

{# spaceless - remove whitespace #}
{% spaceless %}
    <p>    No spaces    </p>
{% endspaceless %}

{# verbatim - don't process template syntax #}
{% verbatim %}
    {{ this won't be processed }}
{% endverbatim %}

{# FILTERS #}

{{ value|add:5 }}              {# Add 5 #}
{{ value|addslashes }}         {# Escape quotes #}
{{ value|capfirst }}           {# Capitalize first letter #}
{{ value|center:15 }}          {# Center in 15 chars #}
{{ value|cut:" " }}            {# Remove spaces #}
{{ value|date:"D d M Y" }}     {# Format date #}
{{ value|default:"n/a" }}      {# Default if falsy #}
{{ value|default_if_none:"n/a" }} {# Default if None #}
{{ value|dictsort:"name" }}    {# Sort list of dicts #}
{{ value|divisibleby:3 }}      {# Check divisibility #}
{{ value|escape }}             {# HTML escape #}
{{ value|filesizeformat }}     {# Human readable size #}
{{ value|first }}              {# First item #}
{{ value|floatformat:2 }}      {# Format float #}
{{ value|get_digit:2 }}        {# Get 2nd digit #}
{{ value|join:", " }}          {# Join list #}
{{ value|json_script:"data" }} {# Safe JSON in script tag #}
{{ value|last }}               {# Last item #}
{{ value|length }}             {# Length #}
{{ value|linebreaks }}         {# Convert \n to <br> #}
{{ value|linenumbers }}        {# Add line numbers #}
{{ value|ljust:10 }}           {# Left justify #}
{{ value|lower }}              {# Lowercase #}
{{ value|make_list }}          {# String to list #}
{{ value|phone2numeric }}      {# Letters to numbers #}
{{ value|pluralize }}          {# Add 's' if plural #}
{{ value|pluralize:"es" }}     {# Custom plural suffix #}
{{ value|random }}             {# Random item #}
{{ value|rjust:10 }}           {# Right justify #}
{{ value|safe }}               {# Mark as safe HTML #}
{{ value|slice:":5" }}         {# Slice list/string #}
{{ value|slugify }}            {# Create slug #}
{{ value|stringformat:"d" }}   {# String format #}
{{ value|striptags }}          {# Remove HTML tags #}
{{ value|time:"H:i" }}         {# Format time #}
{{ value|timesince }}          {# Time since #}
{{ value|timeuntil }}          {# Time until #}
{{ value|title }}              {# Title Case #}
{{ value|truncatechars:25 }}   {# Truncate chars #}
{{ value|truncatewords:10 }}   {# Truncate words #}
{{ value|upper }}              {# Uppercase #}
{{ value|urlencode }}          {# URL encode #}
{{ value|urlize }}             {# Convert URLs to links #}
{{ value|wordcount }}          {# Word count #}
{{ value|wordwrap:75 }}        {# Wrap at 75 chars #}
{{ value|yesno:"yes,no,maybe" }} {# Yes/no/none mapping #}
```

**Custom template tags and filters:**

```python
# myapp/templatetags/myapp_extras.py
from django import template
from django.utils.safestring import mark_safe

register = template.Library()

# Simple filter
@register.filter
def multiply(value, arg):
    return value * arg

# Filter with name
@register.filter(name='cut_spaces')
def cut_spaces(value):
    return value.replace(' ', '')

# Simple tag
@register.simple_tag
def current_time(format_string):
    from datetime import datetime
    return datetime.now().strftime(format_string)

# Simple tag with context
@register.simple_tag(takes_context=True)
def greeting(context):
    user = context['user']
    return f"Hello, {user.username}!"

# Inclusion tag (renders a template)
@register.inclusion_tag('myapp/sidebar.html')
def show_sidebar(num_items=5):
    items = Item.objects.all()[:num_items]
    return {'items': items}

# Usage in template:
# {% load myapp_extras %}
# {{ value|multiply:5 }}
# {% current_time "%Y-%m-%d" %}
# {% show_sidebar 10 %}
```

<a id="q18"></a>
### Q18: How do you use template inheritance?
**Answer:**

```django
{# base.html - Parent template #}
<!DOCTYPE html>
<html>
<head>
    <title>{% block title %}My Site{% endblock %}</title>
    {% block extra_css %}{% endblock %}
</head>
<body>
    <header>
        {% block header %}
        <nav>
            <a href="/">Home</a>
            <a href="/about/">About</a>
        </nav>
        {% endblock %}
    </header>
    
    <main>
        {% block content %}{% endblock %}
    </main>
    
    <aside>
        {% block sidebar %}
        <h3>Sidebar</h3>
        {% endblock %}
    </aside>
    
    <footer>
        {% block footer %}
        <p>&copy; 2024 My Site</p>
        {% endblock %}
    </footer>
    
    {% block extra_js %}{% endblock %}
</body>
</html>

{# articles/list.html - Child template #}
{% extends "base.html" %}

{% block title %}Articles - {{ block.super }}{% endblock %}

{% block content %}
<h1>All Articles</h1>

{% for article in articles %}
    <article>
        <h2>{{ article.title }}</h2>
        <p>{{ article.excerpt }}</p>
    </article>
{% empty %}
    <p>No articles yet.</p>
{% endfor %}
{% endblock %}

{% block sidebar %}
{{ block.super }}  {# Include parent's sidebar content #}
<h4>Categories</h4>
<ul>
{% for category in categories %}
    <li>{{ category.name }}</li>
{% endfor %}
</ul>
{% endblock %}

{# Three-level inheritance #}
{# base.html -> articles/base.html -> articles/detail.html #}

{# articles/base.html #}
{% extends "base.html" %}

{% block content %}
<div class="articles-container">
    {% block articles_content %}{% endblock %}
</div>
{% endblock %}

{# articles/detail.html #}
{% extends "articles/base.html" %}

{% block articles_content %}
<article>
    <h1>{{ article.title }}</h1>
    <div>{{ article.content|safe }}</div>
</article>
{% endblock %}
```

---

## Configuration & Middleware

<a id="q19"></a>
### Q19: What is Django middleware?
**Answer:**

Middleware is a framework of hooks into Django's request/response processing.

```python
# Middleware execution order:
# Request:  SecurityMiddleware → SessionMiddleware → ... → View
# Response: View → ... → SessionMiddleware → SecurityMiddleware

# settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
    'myapp.middleware.CustomMiddleware',  # Custom middleware
]

# Custom middleware (function-based)
def simple_middleware(get_response):
    # One-time configuration and initialization
    
    def middleware(request):
        # Code executed before view (and later middleware)
        print(f"Before view: {request.path}")
        
        response = get_response(request)
        
        # Code executed after view
        print(f"After view: {response.status_code}")
        
        return response
    
    return middleware

# Custom middleware (class-based)
class CustomMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
        # One-time configuration
    
    def __call__(self, request):
        # Before view
        request.custom_attribute = 'value'
        
        response = self.get_response(request)
        
        # After view
        response['X-Custom-Header'] = 'value'
        
        return response
    
    def process_view(self, request, view_func, view_args, view_kwargs):
        # Called just before view
        # Return None to continue, or HttpResponse to short-circuit
        return None
    
    def process_exception(self, request, exception):
        # Called when view raises exception
        # Return None to propagate, or HttpResponse to handle
        return None
    
    def process_template_response(self, request, response):
        # Called after view if response has render() method
        return response

# Practical examples:

# Request timing middleware
import time

class RequestTimingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        start_time = time.time()
        response = self.get_response(request)
        duration = time.time() - start_time
        response['X-Request-Duration'] = f"{duration:.3f}s"
        return response

# IP blocking middleware
class IPBlockMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
        self.blocked_ips = ['192.168.1.100']
    
    def __call__(self, request):
        ip = request.META.get('REMOTE_ADDR')
        if ip in self.blocked_ips:
            from django.http import HttpResponseForbidden
            return HttpResponseForbidden("Blocked")
        return self.get_response(request)

# Maintenance mode middleware
class MaintenanceMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        from django.conf import settings
        if getattr(settings, 'MAINTENANCE_MODE', False):
            from django.shortcuts import render
            return render(request, 'maintenance.html', status=503)
        return self.get_response(request)
```

<a id="q20"></a>
### Q20: How do you configure Django settings for different environments?
**Answer:**

```python
# Method 1: Multiple settings files
# settings/
#   __init__.py
#   base.py
#   development.py
#   production.py
#   testing.py

# settings/base.py
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent.parent

INSTALLED_APPS = [
    'django.contrib.admin',
    # ...
]

# Shared settings...

# settings/development.py
from .base import *

DEBUG = True
ALLOWED_HOSTS = ['localhost', '127.0.0.1']

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}

# settings/production.py
from .base import *
import os

DEBUG = False
ALLOWED_HOSTS = ['mysite.com', 'www.mysite.com']

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME'),
        'USER': os.environ.get('DB_USER'),
        'PASSWORD': os.environ.get('DB_PASSWORD'),
        'HOST': os.environ.get('DB_HOST'),
        'PORT': os.environ.get('DB_PORT', '5432'),
    }
}

# Security settings
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True

# Run with specific settings
# python manage.py runserver --settings=myproject.settings.development
# Or set environment variable: DJANGO_SETTINGS_MODULE=myproject.settings.production

# Method 2: Environment variables with python-decouple
# pip install python-decouple

from decouple import config, Csv

DEBUG = config('DEBUG', default=False, cast=bool)
SECRET_KEY = config('SECRET_KEY')
ALLOWED_HOSTS = config('ALLOWED_HOSTS', cast=Csv())

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': config('DB_NAME'),
        'USER': config('DB_USER'),
        'PASSWORD': config('DB_PASSWORD'),
        'HOST': config('DB_HOST', default='localhost'),
        'PORT': config('DB_PORT', default='5432'),
    }
}

# .env file
"""
DEBUG=True
SECRET_KEY=your-secret-key-here
ALLOWED_HOSTS=localhost,127.0.0.1
DB_NAME=mydb
DB_USER=myuser
DB_PASSWORD=mypassword
DB_HOST=localhost
"""

# Method 3: django-environ
# pip install django-environ

import environ

env = environ.Env(
    DEBUG=(bool, False)
)

environ.Env.read_env(BASE_DIR / '.env')

DEBUG = env('DEBUG')
SECRET_KEY = env('SECRET_KEY')
DATABASES = {
    'default': env.db(),  # Reads DATABASE_URL
}

# .env with DATABASE_URL
"""
DEBUG=True
SECRET_KEY=your-secret-key
DATABASE_URL=postgres://user:password@localhost:5432/dbname
"""

# Method 4: Different settings in __init__.py
# settings/__init__.py
import os

environment = os.environ.get('DJANGO_ENV', 'development')

if environment == 'production':
    from .production import *
elif environment == 'testing':
    from .testing import *
else:
    from .development import *
```

---

[← Python Advanced](python-advanced.md) | [Back to Python Index](README.md) | [Django Advanced →](django-advanced.md)
