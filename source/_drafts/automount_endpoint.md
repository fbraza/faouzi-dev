# Eliminating API Endpoint Boilerplate: Auto-Mounting in Python Web Frameworks

## The Problem: Repetitive Endpoint Wiring

When building APIs with modern Python web frameworks like FastAPI, developers often find themselves writing repetitive boilerplate code. For each business function, you need to:

1. **Define the business logic**
2. **Manually wire HTTP endpoints** 
3. **Handle parameter extraction and type conversion**
4. **Maintain consistency between function signatures and HTTP routes**

This creates a significant amount of repetitive code:

```python
# Business logic
def get_user(user_id: int) -> User:
    return database.find_user(user_id)

def create_user(name: str, email: str) -> User:
    return database.create_user(name, email)

# Manual endpoint wiring (boilerplate!)
@app.get("/get_user")
def get_user_endpoint(user_id: int):
    return get_user(user_id)

@app.post("/create_user") 
def create_user_endpoint(request: CreateUserRequest):
    return create_user(request.name, request.email)
```

## The Core Issues

### 1. **Duplication of Intent**
Every function requires both a business logic definition and a separate HTTP endpoint definition, even when the mapping is straightforward.

### 2. **Manual Type Conversion**
HTTP parameters arrive as strings, requiring manual conversion to the expected types (int, float, bool, etc.).

### 3. **Inconsistent Patterns**
Different endpoints may handle parameter extraction differently, leading to inconsistencies across the API.

### 4. **Maintenance Overhead**
Changes to function signatures require updating both the business logic and the endpoint wrapper.

## The Solution: Automatic Endpoint Mounting

The solution is to eliminate the manual wiring step entirely through **automatic endpoint mounting**. Here's how it works:

### 1. **Procedure Registration**
Instead of manually creating endpoints, functions are registered as "procedures" with metadata about their intended HTTP method:

```python
@router.query()  # Indicates this should be a GET endpoint
def get_user(user_id: int) -> User:
    return database.find_user(user_id)

@router.mutation()  # Indicates this should be a POST endpoint  
def create_user(name: str, email: str) -> User:
    return database.create_user(name, email)
```

### 2. **Automatic Endpoint Generation**
A single method call analyzes all registered procedures and automatically creates the appropriate HTTP endpoints:

```python
router.mount(app)  # Creates all endpoints automatically
```

This automatically generates:
- `GET /get_user?user_id=42` 
- `POST /create_user` (with JSON body)

### 3. **Intelligent Parameter Handling**

The auto-mounting system:

- **Extracts function signatures** using Python's `inspect` module
- **Maps query parameters** for GET requests (queries)
- **Maps JSON body parameters** for POST requests (mutations)
- **Automatically converts types** based on function annotations

```python
# Query: GET /get_user?user_id=42
# Automatically converts "42" (string) → 42 (int)

# Mutation: POST /create_user 
# Body: {"name": "Alice", "email": "alice@example.com"}
# Maps directly to function parameters
```

## Implementation Architecture

### Core Components

**1. Procedure Registry**
```python
self.proc = {
    "get_user": {
        "type": "query",           # HTTP method category
        "handler": function_ref,   # Original function
        "signature": inspect_sig,  # Parameter information
    }
}
```

**2. Dynamic Endpoint Creation**
```python
def mount(self, app: FastAPI, prefix: str = "") -> None:
    for proc_name, proc_data in self.proc.items():
        endpoint_path = f"{prefix}/{proc_name}"
        
        if proc_data["type"] == "query":
            self._create_query_endpoint(app, endpoint_path, proc_data)
        elif proc_data["type"] == "mutation":
            self._create_mutation_endpoint(app, endpoint_path, proc_data)
```

**3. Parameter Conversion Engine**
```python
def _convert_params(self, params: dict, signature: inspect.Signature) -> dict:
    converted = {}
    for param_name, param in signature.parameters.items():
        if param_name in params:
            value = params[param_name]
            param_type = param.annotation
            
            # Type-specific conversion
            if param_type == int:
                converted[param_name] = int(value)
            elif param_type == float:
                converted[param_name] = float(value)
            # ... handle other types
            
    return converted
```

## Benefits Achieved

### 1. **Eliminated Boilerplate**
- **Before**: ~6-10 lines per endpoint (function + wrapper + route)
- **After**: ~3 lines per endpoint (function + decorator only)
- **Reduction**: 50-70% less code

### 2. **Consistent Patterns**
All endpoints follow identical patterns:
- Queries always use query parameters
- Mutations always use JSON bodies
- Type conversion is standardized

### 3. **Zero Configuration** 
No manual route definitions, no parameter extraction logic, no type conversion code.

### 4. **Maintainability**
Function signature changes automatically propagate to HTTP endpoints without manual updates.

## Advanced Features

### Custom Prefixes
```python
router.mount(app, prefix="/api/v1")
# Creates: GET /api/v1/get_user, POST /api/v1/create_user
```

### Default Parameter Handling
```python
def search_users(query: str, limit: int = 10) -> List[User]:
    # GET /search_users?query=alice
    # limit automatically defaults to 10 if not provided
```

### Type Safety Preservation
All original type hints are preserved, enabling:
- IDE autocomplete
- Static type checking
- Runtime validation
- Automatic API documentation

## Comparison with Alternatives

| Approach | Lines of Code | Type Safety | Consistency | Maintenance |
|----------|---------------|-------------|-------------|-------------|
| Manual FastAPI | High | Manual | Variable | High |
| Auto-mounting | Low | Automatic | Enforced | Low |
| GraphQL | Medium | Schema-based | Enforced | Medium |
| REST Frameworks | Medium | Variable | Variable | Medium |

## Conclusion

Automatic endpoint mounting eliminates a significant source of boilerplate in API development while maintaining type safety and improving consistency. By leveraging Python's introspection capabilities and decorator patterns, we can reduce repetitive code by 50-70% while making APIs more maintainable and less error-prone.

This approach transforms API development from "define business logic + wire endpoints" to simply "define business logic" - letting the framework handle the mechanical work of HTTP integration automatically.

The result is code that's more focused on business value and less on framework mechanics, while maintaining all the benefits of type safety and clear API contracts.