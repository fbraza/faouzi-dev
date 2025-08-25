# Understanding Mock Objects in Python Testing: A Deep Dive into Client-Server Testing

When building applications that communicate with external services, testing can become complex. How do you test your HTTP client without actually making network requests? How do you ensure your tests are fast, reliable, and don't depend on external services being available? The answer lies in **mocking**.

In this post, we'll explore how to use Python's `unittest.mock` library to test client-server interactions effectively, using a practical example from a type-safe RPC library.

## The Challenge: Testing HTTP Clients

Let's say you have a client class that fetches data from a remote server:

```python
import requests

class Client:
    def __init__(self, base_url: str):
        self.base_url = base_url
        self.router = requests.get(f"{self.base_url}/schema").json()

    def get(self, endpoint: str, **kwargs):
        schema = self.router[endpoint]["schema"]
        # ... validation logic ...
        response = requests.get(f"{self.base_url}/{endpoint}", params=kwargs)
        return response.json()
```

Testing this client presents several challenges:
- **Network dependency**: Tests require a running server
- **Slow execution**: Network calls add latency
- **Flaky tests**: Network issues can cause random failures
- **Test isolation**: Tests might interfere with each other

## Enter Mock Objects

Mock objects solve these problems by creating "fake" versions of external dependencies. Instead of making real HTTP requests, we can simulate the behavior and control the responses.

## Understanding the `@patch` Decorator

Let's break down this test example:

```python
@patch('requests.get')
def test_client_initialization(mock_get, mock_schema):
    mock_get.return_value.json.return_value = mock_schema
    
    client = Client("http://localhost:8000")
    
    mock_get.assert_called_once_with("http://localhost:8000/schema")
    assert client.router == mock_schema
```

### What is `mock_get`?

`mock_get` is a **mock object** created by the `@patch` decorator. Here's what happens step by step:

### 1. The `@patch` Decorator Magic

```python
@patch('requests.get')
def test_client_initialization(mock_get, mock_schema):
```

This decorator performs three key actions:
1. **Replaces `requests.get`** with a mock object during the test
2. **Passes the mock object** as the first parameter (`mock_get`) to your test function
3. **Restores the original `requests.get`** after the test completes

### 2. Setting Up Mock Return Values

```python
mock_get.return_value.json.return_value = mock_schema
```

This line sets up a **chain of mock returns**. Let's visualize this:

```
mock_get                           ← The mock that replaces requests.get()
├── return_value                   ← The mock HTTP response object
    └── json                       ← The mock JSON method on the response
        └── return_value           ← What the JSON method returns (mock_schema)
```

### 3. The Actual Execution Flow

When your code runs:

```python
# Your actual code does this:
response = requests.get("http://localhost:8000/schema")  # ← This is now mock_get
data = response.json()                                   # ← This returns mock_schema
```

With mocking, the flow becomes:

```python
# Step 1: requests.get() is replaced by mock_get
response = mock_get("http://localhost:8000/schema")      # Returns mock_get.return_value  

# Step 2: response.json() calls the mock json method
data = response.json()                                   # Returns mock_schema
```

## Complete Example with Context

Let's see a full working example:

```python
import pytest
from unittest.mock import patch, Mock
from stitch.client import Client
from pydantic import BaseModel

class User(BaseModel):
    id: int
    name: str
    age: int

@pytest.fixture
def mock_schema():
    return {
        "get_user": {
            "type": "query",
            "schema": {
                "input": {
                    "properties": {"user_id": {"type": "integer"}},
                    "required": ["user_id"]
                },
                "output": {"$ref": "#/defs/User"},
                "$defs": {
                    "User": {
                        "properties": {
                            "id": {"type": "integer"},
                            "name": {"type": "string"},
                            "age": {"type": "integer"}
                        },
                        "required": ["id", "name", "age"]
                    }
                }
            }
        }
    }

@patch('requests.get')
def test_client_initialization(mock_get, mock_schema):
    # 1. Set up what the mock should return
    mock_get.return_value.json.return_value = mock_schema
    
    # 2. When Client() calls requests.get(), it gets the mock instead
    client = Client("http://localhost:8000")
    # Internally: requests.get() → mock_get → mock response → .json() → mock_schema
    
    # 3. Verify the mock was called correctly
    mock_get.assert_called_once_with("http://localhost:8000/schema")
    
    # 4. Verify the client got the mocked data
    assert client.router == mock_schema
```

## Alternative: More Explicit Mock Setup

For better readability, you might prefer this more explicit approach:

```python
@patch('requests.get')
def test_client_initialization_explicit(mock_get, mock_schema):
    # Create a mock response object
    mock_response = Mock()
    mock_response.json.return_value = mock_schema
    
    # Set up mock_get to return our mock response
    mock_get.return_value = mock_response
    
    client = Client("http://localhost:8000")
    
    mock_get.assert_called_once_with("http://localhost:8000/schema")
    assert client.router == mock_schema
```

## Testing Different Scenarios

### Testing Successful API Calls

```python
@patch('requests.get')
def test_successful_api_call(mock_get, mock_schema):
    # Mock both schema fetch and API call
    mock_get.side_effect = [
        Mock(json=lambda: mock_schema),  # Schema fetch
        Mock(json=lambda: {"id": 1, "name": "John", "age": 30})  # API call
    ]
    
    client = Client("http://localhost:8000")
    result = client.get("get_user", user_id=1)
    
    assert result["name"] == "John"
    assert mock_get.call_count == 2  # Called twice
```

### Testing Error Conditions

```python
@patch('requests.get')
def test_network_error(mock_get, mock_schema):
    # Mock schema fetch success, then network error
    mock_get.side_effect = [
        Mock(json=lambda: mock_schema),  # Schema fetch succeeds
        requests.ConnectionError("Network error")  # API call fails
    ]
    
    client = Client("http://localhost:8000")
    
    with pytest.raises(requests.ConnectionError):
        client.get("get_user", user_id=1)
```

### Testing Validation Logic

```python
@patch('requests.get')
def test_missing_required_field(mock_get, mock_schema):
    mock_get.return_value.json.return_value = mock_schema
    
    client = Client("http://localhost:8000")
    
    with pytest.raises(ValueError, match="Missing required field"):
        client.get("get_user")  # Missing user_id parameter
```

## Mock Assertions: Verifying Interactions

Mock objects provide powerful assertion methods to verify how they were used:

```python
# Verify the mock was called
mock_get.assert_called_once()

# Verify it was called with specific arguments
mock_get.assert_called_once_with("http://localhost:8000/schema")

# Verify it was called a specific number of times
mock_get.assert_called()  # At least once
assert mock_get.call_count == 1

# Check call arguments
print(mock_get.call_args)       # Last call arguments
print(mock_get.call_args_list)  # All calls

# Verify it was never called
mock_get.assert_not_called()
```

## Advanced Mocking Patterns

### Using `side_effect` for Dynamic Responses

```python
def dynamic_response(url):
    if "schema" in url:
        return Mock(json=lambda: mock_schema)
    elif "users" in url:
        return Mock(json=lambda: {"id": 1, "name": "John"})
    else:
        raise ValueError("Unknown endpoint")

mock_get.side_effect = dynamic_response
```

### Context Managers for Temporary Mocking

```python
def test_with_context_manager():
    with patch('requests.get') as mock_get:
        mock_get.return_value.json.return_value = {"test": "data"}
        # Test code here
        # Mock is automatically restored after the block
```

### Multiple Patches

```python
@patch('requests.post')
@patch('requests.get')
def test_multiple_methods(mock_get, mock_post, mock_schema):
    # Note: decorators are applied bottom-up, so mock_get comes first
    mock_get.return_value.json.return_value = mock_schema
    mock_post.return_value.json.return_value = {"success": True}
    
    # Test code using both GET and POST
```

## Best Practices

### 1. Mock at the Right Level
```python
# ✅ Good: Mock the external dependency
@patch('requests.get')

# ❌ Avoid: Mocking your own code
@patch('mymodule.Client.get')
```

### 2. Use Fixtures for Reusable Mock Data
```python
@pytest.fixture
def mock_user_response():
    return {"id": 1, "name": "John", "age": 30}

@patch('requests.get')
def test_user_fetch(mock_get, mock_user_response):
    mock_get.return_value.json.return_value = mock_user_response
    # Test code here
```

### 3. Test Both Success and Failure Cases
```python
def test_success_case():
    # Mock successful response
    pass

def test_network_error():
    # Mock network failure
    pass

def test_invalid_response():
    # Mock malformed JSON
    pass
```

### 4. Keep Mock Setup Close to Test Logic
```python
def test_client_behavior():
    with patch('requests.get') as mock_get:
        # Setup mock right before use
        mock_get.return_value.json.return_value = expected_data
        
        # Test immediately
        result = client.fetch_data()
        assert result == expected_data
```

## Common Pitfalls and How to Avoid Them

### 1. Patching the Wrong Import
```python
# If your code does: from requests import get
# Then patch like this:
@patch('mymodule.get')  # Not 'requests.get'

# If your code does: import requests
# Then patch like this:
@patch('requests.get')
```

### 2. Mock Return Chain Confusion
```python
# Wrong: This doesn't work for chained calls
mock_get.return_value = {"data": "test"}

# Right: For response.json() pattern
mock_get.return_value.json.return_value = {"data": "test"}
```

### 3. Forgetting to Assert Mock Calls
```python
def test_client():
    with patch('requests.get') as mock_get:
        mock_get.return_value.json.return_value = test_data
        
        client.fetch_data()
        
        # Don't forget to verify the interaction!
        mock_get.assert_called_once_with(expected_url)
```

## Understanding `side_effect`: Multiple Mock Responses

The `side_effect` property is a powerful feature that allows you to specify different return values for **multiple consecutive calls** to the same mocked function. Let's break down what's happening in a more complex test scenario:

### The Problem: Multiple HTTP Calls

Your `Client` class makes **two separate** HTTP requests:

1. **During initialization**: `requests.get(f"{self.base_url}/schema")` to fetch the schema
2. **During the `get()` call**: `requests.get(f"{self.base_url}/{endpoint}")` to fetch actual data

Since both calls use `requests.get()`, you need to mock **different responses** for each call.

### The `side_effect` Solution

```python
@patch('requests.get')
def test_valid_request(mock_get, mock_schema):
    # Mock schema fetch and API call
    mock_get.side_effect = [
        Mock(json=lambda: mock_schema),  # Schema fetch (1st call)
        Mock(json=lambda: {"id": 1, "name": "John", "age": 30})  # API call (2nd call)
    ]

    client = Client("http://localhost:8000")  # ← 1st requests.get() call
    result = client.get("get_user", user_id=1)  # ← 2nd requests.get() call

    assert result["name"] == "John"
```

### How `side_effect` Works

When you set `mock_get.side_effect` to a **list**, each call to `mock_get()` returns the **next item** in the list:

```python
mock_get.side_effect = [response1, response2, response3]

# 1st call: mock_get() returns response1
# 2nd call: mock_get() returns response2  
# 3rd call: mock_get() returns response3
# 4th call: raises StopIteration (no more items)
```

### Step-by-Step Execution

Let's trace through your test execution:

```python
# 1. Test setup
mock_get.side_effect = [
    Mock(json=lambda: mock_schema),                        # Item 0
    Mock(json=lambda: {"id": 1, "name": "John", "age": 30})  # Item 1
]

# 2. Client initialization
client = Client("http://localhost:8000")
# Internally calls: requests.get("http://localhost:8000/schema")
# mock_get() returns: Mock(json=lambda: mock_schema)  ← Uses item 0
# client.router gets: mock_schema

# 3. API call
result = client.get("get_user", user_id=1) 
# Internally calls: requests.get("http://localhost:8000/get_user", params={"user_id": 1})
# mock_get() returns: Mock(json=lambda: {"id": 1, "name": "John", "age": 30})  ← Uses item 1
# result gets: {"id": 1, "name": "John", "age": 30}
```

### Alternative Ways to Use `side_effect`

#### 1. Using a Function
```python
def dynamic_response(url, **kwargs):
    if "schema" in url:
        return Mock(json=lambda: mock_schema)
    elif "get_user" in url:
        return Mock(json=lambda: {"id": 1, "name": "John", "age": 30})
    else:
        raise ValueError(f"Unexpected URL: {url}")

mock_get.side_effect = dynamic_response
```

#### 2. Using Exceptions
```python
mock_get.side_effect = [
    Mock(json=lambda: mock_schema),  # 1st call succeeds
    requests.ConnectionError("Network error")  # 2nd call fails
]
```

#### 3. Mixing Values and Exceptions
```python
mock_get.side_effect = [
    Mock(json=lambda: mock_schema),          # Success
    requests.Timeout("Request timeout"),     # Timeout error
    Mock(json=lambda: {"retry": "success"})  # Retry succeeds
]
```

### Verifying Multiple Calls

You can verify that multiple calls were made with the expected arguments:

```python
@patch('requests.get')
def test_multiple_calls_verification(mock_get, mock_schema):
    mock_get.side_effect = [
        Mock(json=lambda: mock_schema),
        Mock(json=lambda: {"id": 1, "name": "John", "age": 30})
    ]

    client = Client("http://localhost:8000")
    result = client.get("get_user", user_id=1)

    # Verify both calls were made
    assert mock_get.call_count == 2
    
    # Verify the specific calls
    expected_calls = [
        call("http://localhost:8000/schema"),
        call("http://localhost:8000/get_user", params={"user_id": 1})
    ]
    mock_get.assert_has_calls(expected_calls)
```

### Common Pitfalls

#### 1. Wrong Number of Items
```python
# If you only provide 1 item but make 2 calls:
mock_get.side_effect = [Mock(json=lambda: mock_schema)]  # Only 1 item

client = Client("http://localhost:8000")  # 1st call: OK
client.get("get_user", user_id=1)         # 2nd call: StopIteration error!
```

#### 2. Order Matters
```python
# Wrong order - schema response for API call, API response for schema
mock_get.side_effect = [
    Mock(json=lambda: {"id": 1, "name": "John"}),  # This goes to schema fetch!
    Mock(json=lambda: mock_schema)                  # This goes to API call!
]
```

### When to Use `side_effect` vs `return_value`

- **Use `return_value`** when the mock is called **once** or should **always return the same thing**
- **Use `side_effect`** when the mock is called **multiple times** and should return **different things**

```python
# Same response every time
mock_get.return_value.json.return_value = same_data

# Different response each time  
mock_get.side_effect = [response1, response2, response3]
```

## Understanding `__new__()` in Fixture-Based Testing

When creating test fixtures, you might encounter this pattern:

```python
@pytest.fixture
def client_with_schema():
    # Create a client without HTTP calls
    client = Client.__new__(Client)  # Skip __init__
    client.base_url = "http://localhost:8000"
    client.router = {...}
    return client
```

### What is `Client.__new__(Client)`?

In Python, object creation happens in two phases:

1. **`__new__()`**: Creates the actual instance (allocates memory)
2. **`__init__()`**: Initializes the instance (sets up attributes)

Normally when you do `Client()`, Python automatically:
1. Calls `__new__()` to create the instance
2. Calls `__init__()` to initialize it

### Why Skip `__init__()` in Tests?

Consider a typical `Client` class:

```python
class Client:
    def __init__(self, base_url: str):
        self.base_url = base_url
        self.router = requests.get(f"{self.base_url}/router").json()  # ← HTTP call!
```

The constructor makes an HTTP request! In fixture-based testing, we want to:
- **Avoid any network calls** - Tests should run offline
- **Manually control the client's state** - Use predetermined test data
- **Set up exactly the data we need** - No surprises or external dependencies

### How It Works: Normal vs Test Instantiation

```python
# Normal instantiation (calls __init__)
client = Client("http://localhost:8000")  # Makes HTTP request!

# Using __new__ (skips __init__)
client = Client.__new__(Client)  # No HTTP request!
# Now manually set the attributes
client.base_url = "http://localhost:8000"
client.router = {"test": "data"}  # Our controlled test data
```

### Visual Flow Comparison

```
Normal flow:
Client("http://localhost:8000")
    ↓
__new__() creates empty instance
    ↓
__init__() runs → makes HTTP request to fetch schema
    ↓
Instance ready with real schema from server

Test fixture flow:
Client.__new__(Client)
    ↓
__new__() creates empty instance
    ↓
(skip __init__ - no HTTP request!)
    ↓
Manually set attributes with test data
    ↓
Instance ready with mock schema
```

### Benefits of This Approach

1. **No Network Dependencies**: Tests run without server
2. **Full Control**: You decide exactly what data the client has
3. **Fast Execution**: No HTTP overhead
4. **Predictable**: Same test data every time

### Alternative Approaches

If skipping `__init__()` feels too "hacky", here are cleaner alternatives:

#### 1. Factory Function Pattern
```python
@pytest.fixture
def make_client():
    def _make_client(schema):
        client = Client.__new__(Client)
        client.base_url = "http://localhost:8000"
        client.router = schema
        return client
    return _make_client

def test_something(make_client):
    client = make_client({"get_user": {...}})
    # Test with client
```

#### 2. Subclass for Testing
```python
class TestClient(Client):
    def __init__(self, base_url: str, schema: dict):
        self.base_url = base_url
        self.router = schema  # Don't fetch, use provided schema

@pytest.fixture
def client_with_schema():
    return TestClient("http://localhost:8000", {"get_user": {...}})
```

#### 3. Dependency Injection
```python
class Client:
    def __init__(self, base_url: str, schema_fetcher=None):
        self.base_url = base_url
        if schema_fetcher:
            self.router = schema_fetcher()
        else:
            self.router = requests.get(f"{self.base_url}/router").json()

# In tests
@pytest.fixture
def client_with_schema():
    return Client("http://localhost:8000", 
                  schema_fetcher=lambda: {"get_user": {...}})
```

#### 4. Builder Pattern
```python
class ClientBuilder:
    def __init__(self):
        self.base_url = None
        self.schema = None
    
    def with_base_url(self, url):
        self.base_url = url
        return self
    
    def with_schema(self, schema):
        self.schema = schema
        return self
    
    def build(self):
        client = Client.__new__(Client)
        client.base_url = self.base_url
        client.router = self.schema
        return client

@pytest.fixture
def client_with_schema():
    return (ClientBuilder()
            .with_base_url("http://localhost:8000")
            .with_schema({"get_user": {...}})
            .build())
```

### When to Use Each Approach

- **`__new__()`**: Quick and dirty, good for simple tests
- **Factory Function**: When you need different schemas for different tests
- **Subclass**: When you want a cleaner API and type safety
- **Dependency Injection**: Best for production code testability
- **Builder**: When you have many optional configuration parameters

The `__new__()` approach is perfectly valid for testing, but consider refactoring your production code to use dependency injection for better testability!

## Dependency Injection: Making Your Code More Testable

One of the best ways to make your code easier to test is through **Dependency Injection** (DI). This design pattern allows you to provide dependencies from the outside rather than creating them internally, making your code more flexible and testable.

### The Problem: Tightly Coupled Code

Consider this typical client implementation:

```python
class Client:
    def __init__(self, base_url: str):
        self.base_url = base_url
        # ↓ Dependency is created INSIDE the class
        self.router = requests.get(f"{self.base_url}/router").json()
```

This code is **tightly coupled** to:
- The `requests` library
- A specific endpoint (`/router`)
- Network availability
- The exact schema fetching mechanism

This tight coupling makes testing difficult because you can't control these dependencies from outside the class.

### The Solution: Dependency Injection

```python
class Client:
    def __init__(self, base_url: str, schema_fetcher=None):
        self.base_url = base_url
        # ↓ Dependency is PROVIDED from outside
        if schema_fetcher:
            self.router = schema_fetcher()
        else:
            # Default behavior for production
            self.router = requests.get(f"{self.base_url}/router").json()
```

Now you can **inject** different behaviors:

```python
# Production usage (no change needed)
client = Client("http://api.example.com")

# Test usage (inject mock fetcher)
client = Client("http://test.com", schema_fetcher=lambda: {"test": "schema"})

# Different test scenario
def error_fetcher():
    raise ConnectionError("Network down")
    
client = Client("http://test.com", schema_fetcher=error_fetcher)
```

### Interface-Based Dependency Injection

For larger applications, using interfaces (protocols in Python) provides even better structure:

```python
from abc import ABC, abstractmethod
from typing import Protocol

# Define the interface
class SchemaFetcher(Protocol):
    def fetch(self, base_url: str) -> dict:
        ...

# Production implementation
class HTTPSchemaFetcher:
    def fetch(self, base_url: str) -> dict:
        return requests.get(f"{base_url}/router").json()

# Test implementation
class MockSchemaFetcher:
    def __init__(self, schema: dict):
        self.schema = schema
    
    def fetch(self, base_url: str) -> dict:
        return self.schema

# Client uses the interface
class Client:
    def __init__(self, base_url: str, fetcher: SchemaFetcher = None):
        self.base_url = base_url
        self.fetcher = fetcher or HTTPSchemaFetcher()
        self.router = self.fetcher.fetch(base_url)
```

Usage becomes clean and explicit:

```python
# Production
client = Client("http://api.example.com")  # Uses HTTPSchemaFetcher by default

# Testing
mock_fetcher = MockSchemaFetcher({"test": "data"})
client = Client("http://test.com", mock_fetcher)

# No need for __new__() tricks or @patch decorators!
```

### Types of Dependency Injection

#### 1. Constructor Injection (Most Common)
```python
class EmailService:
    def __init__(self, smtp_client):
        self.smtp = smtp_client  # Injected via constructor
    
    def send(self, message):
        self.smtp.send_mail(message)

# Usage
smtp = SMTPClient("smtp.gmail.com")
service = EmailService(smtp)  # Inject dependency
```

#### 2. Setter Injection
```python
class EmailService:
    def __init__(self):
        self.smtp = None
    
    def set_smtp_client(self, smtp_client):
        self.smtp = smtp_client  # Injected via setter
    
    def send(self, message):
        if not self.smtp:
            raise ValueError("SMTP client not configured")
        self.smtp.send_mail(message)

# Usage
service = EmailService()
service.set_smtp_client(SMTPClient("smtp.gmail.com"))
```

#### 3. Method Injection
```python
class DataProcessor:
    def process(self, data, validator=None):
        # Dependency injected per method call
        validator = validator or DefaultValidator()
        if validator.is_valid(data):
            return self.transform(data)
        raise ValueError("Invalid data")

# Usage
processor = DataProcessor()
processor.process(data, validator=MockValidator())  # Inject for this call
```

#### 4. Property Injection
```python
class Service:
    def __init__(self):
        self._logger = None
    
    @property
    def logger(self):
        if not self._logger:
            self._logger = DefaultLogger()  # Lazy default
        return self._logger
    
    @logger.setter
    def logger(self, logger):
        self._logger = logger  # Inject via property

# Usage
service = Service()
service.logger = MockLogger()  # Inject test logger
```

### Real-World Example: Database Connection

Before (Tightly Coupled):
```python
import psycopg2

class UserRepository:
    def __init__(self):
        # Creates its own connection - hard to test!
        self.conn = psycopg2.connect(
            host="localhost",
            database="production_db",
            user="admin",
            password="secret"
        )
    
    def get_user(self, user_id):
        cursor = self.conn.cursor()
        cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
        return cursor.fetchone()
```

After (With Dependency Injection):
```python
class UserRepository:
    def __init__(self, connection):
        # Connection is injected - easy to test!
        self.conn = connection
    
    def get_user(self, user_id):
        cursor = self.conn.cursor()
        cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
        return cursor.fetchone()

# Production
prod_conn = psycopg2.connect(host="localhost", database="production_db")
repo = UserRepository(prod_conn)

# Testing
mock_conn = Mock()
mock_cursor = Mock()
mock_cursor.fetchone.return_value = ("user1", "John Doe")
mock_conn.cursor.return_value = mock_cursor

test_repo = UserRepository(mock_conn)
user = test_repo.get_user(1)
assert user == ("user1", "John Doe")
```

### Benefits of Dependency Injection

1. **Testability**: Easy to provide test doubles (mocks, stubs, fakes)
2. **Flexibility**: Can swap implementations without changing the class
3. **Decoupling**: Classes don't know HOW dependencies work, just WHAT they do
4. **Single Responsibility**: Classes focus on their core logic
5. **Configuration**: Can configure behavior from outside

### When to Use Dependency Injection

Use DI when your class depends on:
- External services (HTTP clients, databases, file systems)
- Complex objects that are expensive to create
- Configuration that might change between environments
- Anything you want to mock in tests

### Common Patterns with DI

#### Factory Pattern with DI
```python
class ClientFactory:
    def __init__(self, schema_fetcher=None):
        self.schema_fetcher = schema_fetcher or HTTPSchemaFetcher()
    
    def create_client(self, base_url: str) -> Client:
        return Client(base_url, self.schema_fetcher)

# Testing
factory = ClientFactory(schema_fetcher=MockSchemaFetcher())
client = factory.create_client("http://test.com")
```

#### Registry Pattern
```python
class ServiceRegistry:
    def __init__(self):
        self.services = {}
    
    def register(self, name: str, service):
        self.services[name] = service
    
    def get(self, name: str):
        return self.services.get(name)

# Setup
registry = ServiceRegistry()
registry.register("email", EmailService(MockSMTP()))
registry.register("database", UserRepository(MockConnection()))

# Usage
email_service = registry.get("email")
```

### Dependency Injection Containers

For large applications, DI containers can manage dependencies automatically:

```python
# Using a simple DI container
from typing import Dict, Any, Type

class DIContainer:
    def __init__(self):
        self._services: Dict[Type, Any] = {}
        self._singletons: Dict[Type, Any] = {}
    
    def register(self, interface: Type, implementation: Any, singleton=False):
        if singleton:
            self._singletons[interface] = implementation
        else:
            self._services[interface] = implementation
    
    def resolve(self, interface: Type):
        if interface in self._singletons:
            return self._singletons[interface]
        return self._services.get(interface)()

# Setup
container = DIContainer()
container.register(SchemaFetcher, HTTPSchemaFetcher)
container.register(SMTPClient, MockSMTPClient, singleton=True)

# Usage
schema_fetcher = container.resolve(SchemaFetcher)
smtp = container.resolve(SMTPClient)  # Always returns same instance
```

### Key Principle: Don't Create What You Can Inject

The fundamental rule of dependency injection is simple: **Don't create dependencies inside your class if you might need to replace them**. Instead, accept them as parameters.

This makes your code:
- More modular
- Easier to test
- More maintainable
- More flexible

By applying dependency injection consistently, you'll find that your need for complex mocking tricks (like using `__new__()` or extensive `@patch` decorators) largely disappears, replaced by clean, straightforward test code.

## Partial Dependency Injection: A Pragmatic Approach

When implementing dependency injection, you don't have to inject everything. Sometimes a **hybrid approach** - injecting critical dependencies while keeping others hard-coded - is the most pragmatic solution.

### The Question: Should Every Dependency Be Injected?

Consider this client with two dependencies:

```python
class Client:
    def __init__(self, base_url: str, fetcher: SchemaFetcher = None):
        self.base_url = base_url
        self.fetcher = fetcher or HTTPSchemaFetcher()
        self.schema = self.fetcher.fetch(base_url)  # Injected dependency ✅
    
    def get(self, endpoint: str, **kwargs):
        # ... validation logic ...
        response = requests.get(f"{self.base_url}/{endpoint}", params=kwargs)  # Direct dependency ⚠️
        return response.json()
```

The `__init__` uses dependency injection for schema fetching, but the `get` method uses `requests` directly. **Is this bad?**

### The Answer: It Depends

**It's not inherently bad.** In fact, partial dependency injection can be a good balance between flexibility and simplicity.

### Problems with Direct Dependencies

1. **Testing requires mocking** - You'll need `@patch('requests.get')` even with DI
2. **Inconsistent design** - Some dependencies injected, others not
3. **Less flexibility** - Can't swap HTTP client implementations
4. **Mixed concerns** - Client both orchestrates and makes HTTP requests

### Benefits of Keeping Direct Dependencies

1. **Simpler code** - Fewer abstractions to maintain
2. **Pragmatic** - If you always use requests, why abstract it?
3. **YAGNI** - You Aren't Gonna Need It (the flexibility)
4. **Clear intent** - Obviously makes HTTP calls

### Three Approaches to Consider

#### Option 1: Keep Direct Dependency (Pragmatic)

```python
class Client:
    def __init__(self, base_url: str, fetcher: SchemaFetcher = None):
        self.base_url = base_url
        self.fetcher = fetcher or HTTPSchemaFetcher()
        self.schema = self.fetcher.fetch(base_url)
    
    def get(self, endpoint: str, **kwargs):
        # Direct use of requests
        response = requests.get(f"{self.base_url}/{endpoint}", params=kwargs)
        return response.json()
```

**Testing approach:**
```python
@patch('requests.get')
def test_get_method(mock_get):
    # Use DI for schema
    client = Client("http://localhost", MockSchemaFetcher(mock_schema))
    
    # Mock the API call
    mock_get.return_value.json.return_value = {"id": 1, "name": "John"}
    
    result = client.get("get_user", user_id=1)
    assert result["name"] == "John"
```

**When to choose:** 
- Schema fetching is the critical initialization step that needs injection
- API calls vary per test and are easy to mock
- You don't need to swap HTTP libraries

#### Option 2: Full Dependency Injection (Consistent)

```python
from typing import Protocol

class RequestMaker(Protocol):
    def get(self, url: str, **kwargs) -> dict:
        ...

class HTTPRequestMaker:
    def get(self, url: str, **kwargs) -> dict:
        response = requests.get(url, **kwargs)
        response.raise_for_status()
        return response.json()

class Client:
    def __init__(self, base_url: str, fetcher=None, request_maker=None):
        self.base_url = base_url
        self.fetcher = fetcher or HTTPSchemaFetcher()
        self.request_maker = request_maker or HTTPRequestMaker()
        self.schema = self.fetcher.fetch(base_url)
    
    def get(self, endpoint: str, **kwargs):
        return self.request_maker.get(
            f"{self.base_url}/{endpoint}", 
            params=kwargs
        )
```

**Testing approach:**
```python
def test_get_method():
    mock_request_maker = Mock()
    mock_request_maker.get.return_value = {"id": 1, "name": "John"}
    
    client = Client(
        "http://localhost",
        MockSchemaFetcher(mock_schema),
        mock_request_maker
    )
    
    result = client.get("get_user", user_id=1)
    assert result["name"] == "John"
    mock_request_maker.get.assert_called_once()
```

**When to choose:**
- You need full control over all external dependencies
- You might swap HTTP client libraries
- You want consistent architecture throughout

#### Option 3: Extend Existing Abstraction (Unified)

```python
class SchemaFetcher(Protocol):
    def fetch_schema(self, base_url: str) -> dict:
        ...
    
    def call_endpoint(self, url: str, **kwargs) -> dict:
        ...

class HTTPSchemaFetcher:
    def fetch_schema(self, base_url: str) -> dict:
        response = requests.get(f"{base_url}/schema")
        return response.json()
    
    def call_endpoint(self, url: str, **kwargs) -> dict:
        response = requests.get(url, **kwargs)
        return response.json()

class Client:
    def __init__(self, base_url: str, fetcher: SchemaFetcher = None):
        self.base_url = base_url
        self.fetcher = fetcher or HTTPSchemaFetcher()
        self.schema = self.fetcher.fetch_schema(base_url)
    
    def get(self, endpoint: str, **kwargs):
        return self.fetcher.call_endpoint(
            f"{self.base_url}/{endpoint}",
            params=kwargs
        )
```

**When to choose:**
- You want one abstraction for all HTTP operations
- The fetcher naturally handles all remote communication
- You prefer fewer injection points

### Real-World Recommendation

**For most cases, Option 1 (partial injection) is perfectly fine!** Here's why:

1. **Schema fetching is critical** - Happens once at initialization, needs to be controlled
2. **API calls are contextual** - Different for each test, easy to mock individually
3. **Maintenance burden is low** - One abstraction is simpler than multiple
4. **Tests remain clear** - Explicit about what's mocked for each test

### The Hybrid Testing Pattern

```python
class TestClient:
    def test_initialization(self):
        # Only test schema fetching with DI
        client = Client("http://localhost", MockSchemaFetcher(schema))
        assert client.schema == schema
    
    @patch('requests.get')
    def test_get_user(self, mock_get):
        # Use DI for schema, mock for API call
        client = Client("http://localhost", MockSchemaFetcher(schema))
        
        mock_get.return_value.json.return_value = {"id": 1, "name": "John"}
        result = client.get("get_user", user_id=1)
        
        assert result["name"] == "John"
        mock_get.assert_called_with(
            "http://localhost/get_user",
            params={"user_id": 1}
        )
    
    @patch('requests.get')
    def test_network_error(self, mock_get):
        # Use DI for schema, simulate network error
        client = Client("http://localhost", MockSchemaFetcher(schema))
        
        mock_get.side_effect = requests.ConnectionError("Network down")
        
        with pytest.raises(requests.ConnectionError):
            client.get("get_user", user_id=1)
```

### Guidelines for Partial Dependency Injection

#### Inject dependencies that:
- Are needed at initialization time
- Represent core configuration or behavior
- Are expensive to create or require cleanup
- Need to be swapped out entirely for testing

#### Keep direct dependencies that:
- Are used in specific methods only
- Vary significantly between test cases
- Are industry-standard (like `requests`)
- Would add complexity without clear benefit if abstracted

### The Pragmatic Principle

> "Make the critical parts flexible, keep the simple parts simple."

Not every dependency needs injection. The goal is testable, maintainable code - not architectural purity. If mocking `requests.get` for some tests while injecting the schema fetcher makes your code simpler and your tests clearer, that's a perfectly valid approach.

### Evolution Path

Start with partial injection and evolve as needed:

```python
# Stage 1: Direct dependencies (hard to test)
class Client:
    def __init__(self, base_url):
        self.schema = requests.get(f"{base_url}/schema").json()

# Stage 2: Inject critical dependency (your current state)
class Client:
    def __init__(self, base_url, fetcher=None):
        self.schema = fetcher.fetch(base_url) if fetcher else default_fetch()

# Stage 3: Full injection (if/when needed)
class Client:
    def __init__(self, base_url, fetcher=None, request_maker=None):
        # All dependencies injected
```

Move to the next stage only when you actually need the flexibility, not because it's "more correct."

## When NOT to Use Mocks

Mocking is powerful, but not always appropriate:

- **Unit tests**: Mock external dependencies ✅
- **Integration tests**: Use real services ✅  
- **End-to-end tests**: Avoid mocking ✅
- **Testing your own logic**: Don't mock what you're testing ❌

## Conclusion

Mock objects are essential for testing applications that interact with external services. They enable you to:

- **Write fast, reliable tests** that don't depend on network conditions
- **Test error conditions** that are hard to reproduce with real services  
- **Isolate your code** to test specific logic without external interference
- **Control the testing environment** completely

The key to effective mocking is understanding the call chain and setting up your mocks to match the actual usage patterns in your code. Start simple with basic mocks, then gradually add complexity as your testing needs grow.

Remember: mock objects are tools to help you test your code's behavior, not replace the need for integration testing with real services. Use them wisely, and they'll make your test suite both more reliable and more maintainable.

## Further Reading

- [Python unittest.mock documentation](https://docs.python.org/3/library/unittest.mock.html)
- [pytest-mock plugin](https://pytest-mock.readthedocs.io/)
- [Testing strategies for microservices](https://martinfowler.com/articles/microservice-testing/)