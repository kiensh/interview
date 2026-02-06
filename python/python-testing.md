# Python Testing Interview Questions

## Table of Contents

### Testing Fundamentals
- [Q1: What are the different types of tests in Python?](#q1)
- [Q2: What is the difference between unittest and pytest?](#q2)
- [Q3: How do you write tests with unittest?](#q3)
- [Q4: How do you write tests with pytest?](#q4)

### Mocking & Fixtures
- [Q5: What is mocking and when should you use it?](#q5)
- [Q6: How do you use pytest fixtures?](#q6)
- [Q7: How do you mock external dependencies?](#q7)

### Django Testing
- [Q8: How do you test Django models and views?](#q8)
- [Q9: How do you test Django REST Framework APIs?](#q9)
- [Q10: What is Factory Boy and how do you use it?](#q10)

---

## Testing Fundamentals

<a id="q1"></a>
### Q1: What are the different types of tests in Python?
**Answer:**

| Type | Scope | Speed | Purpose |
|------|-------|-------|---------|
| Unit Tests | Single function/method | Fast | Test isolated logic |
| Integration Tests | Multiple components | Medium | Test component interaction |
| End-to-End (E2E) | Entire system | Slow | Test user workflows |
| Functional Tests | Feature/behavior | Medium | Test requirements |

```python
# Unit Test - tests a single function in isolation
def add(a, b):
    return a + b

def test_add():
    assert add(2, 3) == 5
    assert add(-1, 1) == 0
    assert add(0, 0) == 0

# Integration Test - tests multiple components working together
def test_user_registration_flow():
    # Tests User model + email service + database
    user = User.objects.create_user('john', 'john@example.com', 'password')
    assert user.email == 'john@example.com'
    assert User.objects.filter(username='john').exists()

# E2E Test - tests complete user workflow
def test_checkout_process(selenium):
    # Login
    selenium.get('/login')
    selenium.find_element_by_name('username').send_keys('user')
    selenium.find_element_by_name('password').send_keys('pass')
    selenium.find_element_by_id('login-btn').click()
    
    # Add to cart
    selenium.get('/products/1')
    selenium.find_element_by_id('add-to-cart').click()
    
    # Checkout
    selenium.get('/checkout')
    selenium.find_element_by_id('place-order').click()
    
    assert 'Order Confirmed' in selenium.page_source

# Test Pyramid
"""
        /\
       /E2E\        <- Few, slow, expensive
      /------\
     /Integra-\     <- Some, medium speed
    /---tion---\
   /-----------\
  /    Unit     \   <- Many, fast, cheap
 /---------------\
"""
```

<a id="q2"></a>
### Q2: What is the difference between unittest and pytest?
**Answer:**

| Feature | unittest | pytest |
|---------|----------|--------|
| Style | Class-based (OOP) | Function-based |
| Assertions | `self.assertEqual()` | `assert` statement |
| Setup/Teardown | `setUp()`, `tearDown()` | Fixtures |
| Test discovery | `test*.py` | `test_*.py`, `*_test.py` |
| Fixtures | Limited | Powerful & flexible |
| Plugins | Few | Extensive ecosystem |
| Output | Basic | Rich, detailed |
| Built-in | Yes (stdlib) | No (pip install) |

```python
# unittest style
import unittest

class TestCalculator(unittest.TestCase):
    def setUp(self):
        self.calc = Calculator()
    
    def tearDown(self):
        pass
    
    def test_add(self):
        self.assertEqual(self.calc.add(2, 3), 5)
    
    def test_divide(self):
        self.assertEqual(self.calc.divide(10, 2), 5)
        with self.assertRaises(ZeroDivisionError):
            self.calc.divide(10, 0)

if __name__ == '__main__':
    unittest.main()

# pytest style
import pytest

@pytest.fixture
def calc():
    return Calculator()

def test_add(calc):
    assert calc.add(2, 3) == 5

def test_divide(calc):
    assert calc.divide(10, 2) == 5

def test_divide_by_zero(calc):
    with pytest.raises(ZeroDivisionError):
        calc.divide(10, 0)

# pytest parametrize
@pytest.mark.parametrize("a,b,expected", [
    (2, 3, 5),
    (-1, 1, 0),
    (0, 0, 0),
])
def test_add_parametrized(calc, a, b, expected):
    assert calc.add(a, b) == expected

# Run tests
# unittest: python -m unittest discover
# pytest: pytest
# pytest with verbose: pytest -v
# pytest specific file: pytest test_calculator.py
# pytest specific test: pytest test_calculator.py::test_add
```

<a id="q3"></a>
### Q3: How do you write tests with unittest?
**Answer:**

```python
import unittest
from unittest.mock import Mock, patch, MagicMock

class TestUserService(unittest.TestCase):
    
    @classmethod
    def setUpClass(cls):
        """Run once before all tests in this class"""
        cls.database = create_test_database()
    
    @classmethod
    def tearDownClass(cls):
        """Run once after all tests in this class"""
        cls.database.destroy()
    
    def setUp(self):
        """Run before each test method"""
        self.service = UserService()
        self.user = User(name='John', email='john@example.com')
    
    def tearDown(self):
        """Run after each test method"""
        self.service.cleanup()
    
    # Basic assertions
    def test_create_user(self):
        user = self.service.create_user('Alice', 'alice@example.com')
        
        self.assertEqual(user.name, 'Alice')
        self.assertNotEqual(user.name, 'Bob')
        self.assertTrue(user.is_active)
        self.assertFalse(user.is_admin)
        self.assertIsNotNone(user.id)
        self.assertIsNone(user.deleted_at)
        self.assertIn('alice', user.email)
        self.assertNotIn('bob', user.email)
        self.assertIsInstance(user, User)
        self.assertGreater(user.id, 0)
        self.assertLess(user.age, 100)
    
    # Testing exceptions
    def test_create_user_invalid_email(self):
        with self.assertRaises(ValueError):
            self.service.create_user('Alice', 'invalid-email')
        
        # Check exception message
        with self.assertRaises(ValueError) as context:
            self.service.create_user('Alice', 'invalid-email')
        self.assertIn('Invalid email', str(context.exception))
    
    # Testing with subTest
    def test_email_validation(self):
        invalid_emails = ['', 'test', '@test.com', 'test@', 'test@.com']
        
        for email in invalid_emails:
            with self.subTest(email=email):
                with self.assertRaises(ValueError):
                    self.service.validate_email(email)
    
    # Skip tests
    @unittest.skip("Feature not implemented yet")
    def test_future_feature(self):
        pass
    
    @unittest.skipIf(sys.version_info < (3, 10), "Requires Python 3.10+")
    def test_python310_feature(self):
        pass
    
    @unittest.expectedFailure
    def test_known_bug(self):
        # This test is expected to fail
        self.assertEqual(buggy_function(), 'correct')

# Test suite
def suite():
    suite = unittest.TestSuite()
    suite.addTest(TestUserService('test_create_user'))
    suite.addTest(TestUserService('test_create_user_invalid_email'))
    return suite

if __name__ == '__main__':
    runner = unittest.TextTestRunner(verbosity=2)
    runner.run(suite())
```

<a id="q4"></a>
### Q4: How do you write tests with pytest?
**Answer:**

```python
import pytest

# Basic test function
def test_addition():
    assert 1 + 1 == 2

# Test class (no inheritance needed)
class TestCalculator:
    def test_add(self):
        assert add(2, 3) == 5
    
    def test_subtract(self):
        assert subtract(5, 3) == 2

# Fixtures
@pytest.fixture
def user():
    return User(name='John', email='john@example.com')

@pytest.fixture
def authenticated_client(user):
    client = Client()
    client.login(user)
    return client

def test_user_profile(authenticated_client, user):
    response = authenticated_client.get(f'/users/{user.id}')
    assert response.status_code == 200

# Fixture scopes
@pytest.fixture(scope='function')  # Default - run for each test
def temp_file():
    f = create_temp_file()
    yield f
    f.delete()

@pytest.fixture(scope='class')  # Run once per test class
def database():
    db = create_database()
    yield db
    db.destroy()

@pytest.fixture(scope='module')  # Run once per module
def config():
    return load_config()

@pytest.fixture(scope='session')  # Run once per test session
def api_client():
    return APIClient()

# Parametrized tests
@pytest.mark.parametrize("input,expected", [
    ("hello", "HELLO"),
    ("World", "WORLD"),
    ("PyTest", "PYTEST"),
])
def test_uppercase(input, expected):
    assert input.upper() == expected

# Multiple parameters
@pytest.mark.parametrize("a,b", [(1, 2), (3, 4)])
@pytest.mark.parametrize("c", [5, 6])
def test_combinations(a, b, c):
    # Tests: (1,2,5), (1,2,6), (3,4,5), (3,4,6)
    pass

# Exception testing
def test_division_by_zero():
    with pytest.raises(ZeroDivisionError):
        1 / 0

def test_exception_message():
    with pytest.raises(ValueError, match=r".*invalid.*"):
        raise ValueError("This is invalid input")

# Markers
@pytest.mark.slow
def test_slow_operation():
    pass

@pytest.mark.skip(reason="Not implemented")
def test_future_feature():
    pass

@pytest.mark.skipif(sys.platform == 'win32', reason="Not supported on Windows")
def test_unix_feature():
    pass

@pytest.mark.xfail(reason="Known bug")
def test_known_failure():
    assert buggy_function() == 'correct'

# Run only marked tests: pytest -m slow
# Skip slow tests: pytest -m "not slow"

# pytest.ini or pyproject.toml configuration
"""
[pytest]
markers =
    slow: marks tests as slow
    integration: marks tests as integration tests
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = -v --tb=short
"""

# conftest.py - shared fixtures
"""
# tests/conftest.py
import pytest

@pytest.fixture
def app():
    from myapp import create_app
    app = create_app('testing')
    yield app

@pytest.fixture
def client(app):
    return app.test_client()
"""
```

---

## Mocking & Fixtures

<a id="q5"></a>
### Q5: What is mocking and when should you use it?
**Answer:**

Mocking replaces real objects with fake ones that you control, useful for:
- External APIs/services
- Database operations
- Time-dependent code
- Expensive operations

```python
from unittest.mock import Mock, MagicMock, patch, call

# Basic Mock
mock = Mock()
mock.method(1, 2, 3)
mock.method.assert_called_once_with(1, 2, 3)

# Configure return value
mock = Mock(return_value=42)
assert mock() == 42

mock.method.return_value = 'result'
assert mock.method() == 'result'

# Side effects
mock = Mock(side_effect=ValueError('Error'))
with pytest.raises(ValueError):
    mock()

# Multiple return values
mock = Mock(side_effect=[1, 2, 3])
assert mock() == 1
assert mock() == 2
assert mock() == 3

# MagicMock - includes magic methods
mock = MagicMock()
mock.__str__.return_value = 'mock string'
assert str(mock) == 'mock string'

# Patching with decorator
@patch('mymodule.requests.get')
def test_api_call(mock_get):
    mock_get.return_value.json.return_value = {'key': 'value'}
    
    result = my_function_that_calls_api()
    
    assert result == {'key': 'value'}
    mock_get.assert_called_once()

# Patching with context manager
def test_api_call():
    with patch('mymodule.requests.get') as mock_get:
        mock_get.return_value.status_code = 200
        mock_get.return_value.json.return_value = {'data': 'test'}
        
        result = fetch_data()
        
        assert result == {'data': 'test'}

# Patching multiple objects
@patch('mymodule.ServiceA')
@patch('mymodule.ServiceB')
def test_with_multiple_mocks(mock_b, mock_a):  # Note: reverse order!
    pass

# patch.object - patch specific attribute
class MyClass:
    def method(self):
        return 'real'

@patch.object(MyClass, 'method', return_value='mocked')
def test_method(mock_method):
    obj = MyClass()
    assert obj.method() == 'mocked'

# Asserting calls
mock = Mock()
mock.method('arg1', key='value')
mock.method('arg2')

mock.method.assert_called()
mock.method.assert_called_once()  # Fails - called twice
mock.method.assert_called_with('arg2')  # Last call
mock.method.assert_any_call('arg1', key='value')  # Any call

# Check call count and calls
assert mock.method.call_count == 2
assert mock.method.call_args_list == [
    call('arg1', key='value'),
    call('arg2'),
]

# Spec - ensure mock has same interface as real object
mock = Mock(spec=RealClass)
mock.real_method()  # OK
mock.fake_method()  # AttributeError

# Auto-spec
@patch('mymodule.MyClass', autospec=True)
def test_with_autospec(MockClass):
    instance = MockClass.return_value
    instance.method.return_value = 'result'
```

<a id="q6"></a>
### Q6: How do you use pytest fixtures?
**Answer:**

```python
import pytest
import tempfile
import os

# Basic fixture
@pytest.fixture
def sample_data():
    return {'name': 'John', 'age': 30}

def test_process_data(sample_data):
    assert sample_data['name'] == 'John'

# Fixture with setup and teardown
@pytest.fixture
def temp_directory():
    # Setup
    dir_path = tempfile.mkdtemp()
    
    yield dir_path  # Test runs here
    
    # Teardown
    import shutil
    shutil.rmtree(dir_path)

def test_file_operations(temp_directory):
    file_path = os.path.join(temp_directory, 'test.txt')
    with open(file_path, 'w') as f:
        f.write('test')
    assert os.path.exists(file_path)

# Fixture that uses other fixtures
@pytest.fixture
def user():
    return User(name='John')

@pytest.fixture
def authenticated_user(user):
    user.authenticate()
    return user

@pytest.fixture
def user_with_posts(authenticated_user):
    authenticated_user.create_post('Hello World')
    return authenticated_user

# Parametrized fixtures
@pytest.fixture(params=['sqlite', 'postgres', 'mysql'])
def database(request):
    db = create_database(request.param)
    yield db
    db.destroy()

def test_database_operations(database):
    # This test runs 3 times with different databases
    database.insert({'key': 'value'})
    assert database.get('key') == 'value'

# Factory fixture
@pytest.fixture
def create_user():
    created_users = []
    
    def _create_user(name, email=None):
        email = email or f'{name.lower()}@example.com'
        user = User(name=name, email=email)
        created_users.append(user)
        return user
    
    yield _create_user
    
    # Cleanup
    for user in created_users:
        user.delete()

def test_multiple_users(create_user):
    user1 = create_user('Alice')
    user2 = create_user('Bob', 'bob@test.com')
    assert user1.name == 'Alice'
    assert user2.email == 'bob@test.com'

# Autouse fixture
@pytest.fixture(autouse=True)
def reset_database():
    yield
    Database.reset()

# conftest.py for shared fixtures
"""
# tests/conftest.py

@pytest.fixture(scope='session')
def app():
    return create_app('testing')

@pytest.fixture(scope='session')
def db(app):
    with app.app_context():
        db.create_all()
        yield db
        db.drop_all()

@pytest.fixture
def client(app):
    return app.test_client()
"""
```

<a id="q7"></a>
### Q7: How do you mock external dependencies?
**Answer:**

```python
from unittest.mock import patch, Mock, MagicMock
import pytest
import requests

# Service to test
class WeatherService:
    def __init__(self, api_key):
        self.api_key = api_key
        self.base_url = 'https://api.weather.com'
    
    def get_temperature(self, city):
        response = requests.get(
            f'{self.base_url}/weather',
            params={'city': city, 'key': self.api_key}
        )
        response.raise_for_status()
        return response.json()['temperature']

# Mocking HTTP requests
class TestWeatherService:
    
    @patch('mymodule.requests.get')
    def test_get_temperature_success(self, mock_get):
        # Configure mock
        mock_response = Mock()
        mock_response.json.return_value = {'temperature': 25}
        mock_response.raise_for_status = Mock()
        mock_get.return_value = mock_response
        
        # Test
        service = WeatherService('fake-key')
        temp = service.get_temperature('London')
        
        # Assertions
        assert temp == 25
        mock_get.assert_called_once_with(
            'https://api.weather.com/weather',
            params={'city': 'London', 'key': 'fake-key'}
        )
    
    @patch('mymodule.requests.get')
    def test_get_temperature_api_error(self, mock_get):
        mock_get.side_effect = requests.RequestException('API Error')
        
        service = WeatherService('fake-key')
        
        with pytest.raises(requests.RequestException):
            service.get_temperature('London')

# Using responses library (cleaner for requests)
# pip install responses
import responses

class TestWeatherServiceWithResponses:
    
    @responses.activate
    def test_get_temperature(self):
        responses.add(
            responses.GET,
            'https://api.weather.com/weather',
            json={'temperature': 25},
            status=200
        )
        
        service = WeatherService('fake-key')
        assert service.get_temperature('London') == 25

# Mocking datetime
from datetime import datetime

class TestTimeDependent:
    
    @patch('mymodule.datetime')
    def test_is_weekend(self, mock_datetime):
        # Mock Saturday
        mock_datetime.now.return_value = datetime(2024, 1, 6)  # Saturday
        mock_datetime.side_effect = lambda *args, **kwargs: datetime(*args, **kwargs)
        
        assert is_weekend() is True

# Using freezegun (better for datetime)
# pip install freezegun
from freezegun import freeze_time

class TestWithFreezegun:
    
    @freeze_time("2024-01-06")  # Saturday
    def test_is_weekend(self):
        assert is_weekend() is True
    
    @freeze_time("2024-01-08")  # Monday
    def test_is_not_weekend(self):
        assert is_weekend() is False

# Mocking environment variables
class TestEnvironmentVariables:
    
    @patch.dict('os.environ', {'API_KEY': 'test-key', 'DEBUG': 'true'})
    def test_with_env_vars(self):
        import os
        assert os.environ['API_KEY'] == 'test-key'
        assert os.environ['DEBUG'] == 'true'

# Mocking database
@pytest.fixture
def mock_db():
    with patch('mymodule.database') as mock:
        mock.query.return_value = [
            {'id': 1, 'name': 'John'},
            {'id': 2, 'name': 'Jane'},
        ]
        yield mock

def test_get_users(mock_db):
    users = get_all_users()
    assert len(users) == 2
    mock_db.query.assert_called_once()
```

---

## Django Testing

<a id="q8"></a>
### Q8: How do you test Django models and views?
**Answer:**

```python
from django.test import TestCase, Client
from django.urls import reverse
from django.contrib.auth.models import User
from .models import Article

# Model tests
class ArticleModelTest(TestCase):
    
    @classmethod
    def setUpTestData(cls):
        """Set up data for the whole TestCase"""
        cls.user = User.objects.create_user('testuser', 'test@example.com', 'password')
        cls.article = Article.objects.create(
            title='Test Article',
            content='Test content',
            author=cls.user
        )
    
    def test_article_creation(self):
        self.assertEqual(self.article.title, 'Test Article')
        self.assertEqual(self.article.author.username, 'testuser')
    
    def test_article_str(self):
        self.assertEqual(str(self.article), 'Test Article')
    
    def test_article_slug_generated(self):
        self.assertEqual(self.article.slug, 'test-article')
    
    def test_get_absolute_url(self):
        expected_url = f'/articles/{self.article.slug}/'
        self.assertEqual(self.article.get_absolute_url(), expected_url)

# View tests
class ArticleViewTest(TestCase):
    
    def setUp(self):
        self.client = Client()
        self.user = User.objects.create_user('testuser', 'test@example.com', 'password')
        self.article = Article.objects.create(
            title='Test Article',
            content='Test content',
            author=self.user,
            status='published'
        )
    
    def test_article_list_view(self):
        response = self.client.get(reverse('article-list'))
        
        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed(response, 'articles/list.html')
        self.assertContains(response, 'Test Article')
        self.assertQuerySetEqual(
            response.context['articles'],
            [self.article]
        )
    
    def test_article_detail_view(self):
        response = self.client.get(
            reverse('article-detail', kwargs={'slug': self.article.slug})
        )
        
        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed(response, 'articles/detail.html')
        self.assertEqual(response.context['article'], self.article)
    
    def test_article_detail_not_found(self):
        response = self.client.get(
            reverse('article-detail', kwargs={'slug': 'nonexistent'})
        )
        self.assertEqual(response.status_code, 404)
    
    def test_article_create_requires_login(self):
        response = self.client.get(reverse('article-create'))
        self.assertRedirects(response, '/login/?next=/articles/create/')
    
    def test_article_create_authenticated(self):
        self.client.login(username='testuser', password='password')
        
        response = self.client.get(reverse('article-create'))
        self.assertEqual(response.status_code, 200)
        
        response = self.client.post(reverse('article-create'), {
            'title': 'New Article',
            'content': 'New content',
        })
        
        self.assertEqual(response.status_code, 302)  # Redirect after success
        self.assertTrue(Article.objects.filter(title='New Article').exists())
    
    def test_article_update_only_author(self):
        other_user = User.objects.create_user('other', 'other@example.com', 'password')
        self.client.login(username='other', password='password')
        
        response = self.client.post(
            reverse('article-update', kwargs={'slug': self.article.slug}),
            {'title': 'Hacked!', 'content': 'Hacked content'}
        )
        
        self.assertEqual(response.status_code, 403)

# Form tests
class ArticleFormTest(TestCase):
    
    def test_valid_form(self):
        form = ArticleForm(data={
            'title': 'Test Title',
            'content': 'Test content here',
        })
        self.assertTrue(form.is_valid())
    
    def test_invalid_form_empty_title(self):
        form = ArticleForm(data={
            'title': '',
            'content': 'Test content',
        })
        self.assertFalse(form.is_valid())
        self.assertIn('title', form.errors)
    
    def test_title_too_short(self):
        form = ArticleForm(data={
            'title': 'Hi',
            'content': 'Test content',
        })
        self.assertFalse(form.is_valid())
        self.assertIn('Title must be at least', str(form.errors['title']))
```

<a id="q9"></a>
### Q9: How do you test Django REST Framework APIs?
**Answer:**

```python
from rest_framework.test import APITestCase, APIClient
from rest_framework import status
from django.urls import reverse
from django.contrib.auth.models import User
from .models import Article

class ArticleAPITest(APITestCase):
    
    def setUp(self):
        self.client = APIClient()
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123'
        )
        self.article = Article.objects.create(
            title='Test Article',
            content='Test content',
            author=self.user,
            status='published'
        )
        self.list_url = reverse('article-list')
        self.detail_url = reverse('article-detail', kwargs={'pk': self.article.pk})
    
    def test_list_articles(self):
        response = self.client.get(self.list_url)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data['results']), 1)
        self.assertEqual(response.data['results'][0]['title'], 'Test Article')
    
    def test_retrieve_article(self):
        response = self.client.get(self.detail_url)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data['title'], 'Test Article')
    
    def test_create_article_unauthenticated(self):
        data = {'title': 'New Article', 'content': 'Content'}
        response = self.client.post(self.list_url, data)
        
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)
    
    def test_create_article_authenticated(self):
        self.client.force_authenticate(user=self.user)
        
        data = {
            'title': 'New Article',
            'content': 'New content here'
        }
        response = self.client.post(self.list_url, data)
        
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(Article.objects.count(), 2)
        self.assertEqual(response.data['title'], 'New Article')
    
    def test_update_article_as_author(self):
        self.client.force_authenticate(user=self.user)
        
        data = {'title': 'Updated Title', 'content': 'Updated content'}
        response = self.client.put(self.detail_url, data)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.article.refresh_from_db()
        self.assertEqual(self.article.title, 'Updated Title')
    
    def test_update_article_as_non_author(self):
        other_user = User.objects.create_user('other', 'other@example.com', 'pass')
        self.client.force_authenticate(user=other_user)
        
        data = {'title': 'Hacked', 'content': 'Hacked'}
        response = self.client.put(self.detail_url, data)
        
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)
    
    def test_delete_article(self):
        self.client.force_authenticate(user=self.user)
        
        response = self.client.delete(self.detail_url)
        
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)
        self.assertEqual(Article.objects.count(), 0)
    
    def test_filter_articles_by_status(self):
        Article.objects.create(
            title='Draft Article',
            content='Draft',
            author=self.user,
            status='draft'
        )
        
        response = self.client.get(self.list_url, {'status': 'published'})
        
        self.assertEqual(len(response.data['results']), 1)
    
    def test_search_articles(self):
        response = self.client.get(self.list_url, {'search': 'Test'})
        
        self.assertEqual(len(response.data['results']), 1)

# Using pytest with DRF
import pytest
from rest_framework.test import APIClient

@pytest.fixture
def api_client():
    return APIClient()

@pytest.fixture
def authenticated_client(api_client, user):
    api_client.force_authenticate(user=user)
    return api_client

@pytest.mark.django_db
def test_create_article(authenticated_client):
    response = authenticated_client.post('/api/articles/', {
        'title': 'Test',
        'content': 'Content'
    })
    assert response.status_code == 201

# Testing with JWT
class JWTAuthTest(APITestCase):
    
    def setUp(self):
        self.user = User.objects.create_user('test', 'test@example.com', 'password')
    
    def test_obtain_token(self):
        response = self.client.post('/api/token/', {
            'username': 'test',
            'password': 'password'
        })
        
        self.assertEqual(response.status_code, 200)
        self.assertIn('access', response.data)
        self.assertIn('refresh', response.data)
    
    def test_access_protected_endpoint_with_token(self):
        # Get token
        response = self.client.post('/api/token/', {
            'username': 'test',
            'password': 'password'
        })
        token = response.data['access']
        
        # Use token
        self.client.credentials(HTTP_AUTHORIZATION=f'Bearer {token}')
        response = self.client.get('/api/articles/')
        
        self.assertEqual(response.status_code, 200)
```

<a id="q10"></a>
### Q10: What is Factory Boy and how do you use it?
**Answer:**

Factory Boy creates test fixtures/objects declaratively, making tests cleaner and more maintainable.

```python
# pip install factory_boy

import factory
from factory.django import DjangoModelFactory
from django.contrib.auth.models import User
from .models import Article, Category, Tag

# Basic factory
class UserFactory(DjangoModelFactory):
    class Meta:
        model = User
    
    username = factory.Sequence(lambda n: f'user{n}')
    email = factory.LazyAttribute(lambda obj: f'{obj.username}@example.com')
    password = factory.PostGenerationMethodCall('set_password', 'password123')
    first_name = factory.Faker('first_name')
    last_name = factory.Faker('last_name')

class CategoryFactory(DjangoModelFactory):
    class Meta:
        model = Category
    
    name = factory.Sequence(lambda n: f'Category {n}')
    slug = factory.LazyAttribute(lambda obj: obj.name.lower().replace(' ', '-'))

class ArticleFactory(DjangoModelFactory):
    class Meta:
        model = Article
    
    title = factory.Faker('sentence', nb_words=5)
    slug = factory.LazyAttribute(lambda obj: obj.title.lower().replace(' ', '-')[:50])
    content = factory.Faker('paragraphs', nb=3)
    author = factory.SubFactory(UserFactory)
    category = factory.SubFactory(CategoryFactory)
    status = 'draft'
    
    # Handle many-to-many relationships
    @factory.post_generation
    def tags(self, create, extracted, **kwargs):
        if not create:
            return
        
        if extracted:
            for tag in extracted:
                self.tags.add(tag)

# Factory with traits
class ArticleFactory(DjangoModelFactory):
    class Meta:
        model = Article
    
    title = factory.Faker('sentence')
    content = factory.Faker('text')
    author = factory.SubFactory(UserFactory)
    status = 'draft'
    
    class Params:
        published = factory.Trait(
            status='published',
            published_at=factory.Faker('date_time_this_year')
        )
        featured = factory.Trait(
            is_featured=True,
            view_count=factory.Faker('random_int', min=1000, max=10000)
        )

# Usage in tests
class TestArticle:
    
    @pytest.fixture
    def user(self):
        return UserFactory()
    
    @pytest.fixture
    def published_article(self, user):
        return ArticleFactory(author=user, published=True)
    
    def test_create_article(self):
        article = ArticleFactory()
        assert article.pk is not None
        assert article.author is not None
    
    def test_create_published_article(self):
        article = ArticleFactory(published=True)
        assert article.status == 'published'
        assert article.published_at is not None
    
    def test_create_featured_published_article(self):
        article = ArticleFactory(published=True, featured=True)
        assert article.status == 'published'
        assert article.is_featured is True
        assert article.view_count >= 1000
    
    def test_create_multiple_articles(self):
        articles = ArticleFactory.create_batch(5, published=True)
        assert len(articles) == 5
        assert all(a.status == 'published' for a in articles)
    
    def test_article_with_tags(self):
        tag1 = TagFactory(name='Python')
        tag2 = TagFactory(name='Django')
        article = ArticleFactory(tags=[tag1, tag2])
        
        assert article.tags.count() == 2
    
    def test_build_without_saving(self):
        # build() creates object without saving to DB
        article = ArticleFactory.build()
        assert article.pk is None
    
    def test_stub_for_mock(self):
        # stub() creates a simple object (not a model instance)
        article = ArticleFactory.stub()
        assert hasattr(article, 'title')

# Integration with pytest-factoryboy
# pip install pytest-factoryboy

# conftest.py
from pytest_factoryboy import register

register(UserFactory)
register(ArticleFactory)

# Now fixtures are auto-registered
def test_with_registered_factory(user, article):
    assert article.author == user  # If configured with subfactory
```

---

[← Django Advanced](django-advanced.md) | [Back to Python Index](README.md)
