# Metaprogramming in Python: Templates vs AST for Code Generation

## Introduction: When You Need to Write Code That Writes Code

Imagine you're building a Python SDK for a REST API with dozens of endpoints. You could manually write wrapper functions for each endpoint, but that's repetitive and error-prone. What if the API changes? You'd need to update dozens of files manually.

This is where **code generation** becomes invaluable. Instead of writing the SDK by hand, you can write code that analyzes the API schema and generates the SDK automatically. This approach ensures consistency, reduces errors, and makes updates trivial.

Python offers two powerful approaches for code generation:
1. **Template-based generation** using Jinja2
2. **AST-based generation** using Python's Abstract Syntax Tree

Each approach has distinct trade-offs. Let's explore both with practical examples.

## The Problem: Building a Typed API Client

Consider this scenario: You have an API schema and need to generate a typed Python client. For a simple API with these endpoints:

```json
{
  "get_user": {
    "method": "GET",
    "params": {"user_id": "integer"},
    "returns": "User"
  },
  "create_post": {
    "method": "POST", 
    "params": {"title": "string", "content": "string", "author_id": "integer"},
    "returns": "Post"
  },
  "list_posts": {
    "method": "GET",
    "params": {"limit": "integer", "offset": "integer"},
    "returns": "List[Post]"
  }
}
```

You want to generate a client like this:

```python
class APIClient:
    def get_user(self, user_id: int) -> User:
        """Get user by ID"""
        ...
    
    def create_post(self, title: str, content: str, author_id: int) -> Post:
        """Create a new post"""
        ...
    
    def list_posts(self, limit: int, offset: int) -> List[Post]:
        """List posts with pagination"""
        ...
```

Let's see how both approaches handle this challenge.

## Approach 1: Template-Based Generation with Jinja2

Template-based generation treats code as text with placeholders. It's intuitive and works well for straightforward transformations.

### Implementation

```python
# template_generator.py
from jinja2 import Template
import json

# The template - looks like Python code with placeholders
CLIENT_TEMPLATE = '''
from typing import List
import requests

class APIClient:
    def __init__(self, base_url: str):
        self.base_url = base_url
        self.session = requests.Session()
    
    {% for method_name, method_info in methods.items() %}
    def {{ method_name }}(self, {% for param, param_type in method_info.params.items() %}{{ param }}: {{ param_type }}{{ ", " if not loop.last }}{% endfor %}) -> {{ method_info.returns }}:
        """{{ method_info.description | default("Auto-generated method") }}"""
        {% if method_info.method == "GET" %}
        response = self.session.get(
            f"{self.base_url}/{{ method_name }}",
            params={
                {% for param in method_info.params.keys() %}
                "{{ param }}": {{ param }},
                {% endfor %}
            }
        )
        {% else %}
        response = self.session.post(
            f"{self.base_url}/{{ method_name }}",
            json={
                {% for param in method_info.params.keys() %}
                "{{ param }}": {{ param }},
                {% endfor %}
            }
        )
        {% endif %}
        response.raise_for_status()
        return response.json()
    
    {% endfor %}
'''

def generate_client_template(schema: dict, output_file: str):
    """Generate client using Jinja2 templates"""
    
    # Transform schema for template
    methods = {}
    for method_name, method_data in schema.items():
        methods[method_name] = {
            "method": method_data["method"],
            "params": {param: map_type(ptype) for param, ptype in method_data["params"].items()},
            "returns": map_type(method_data["returns"]),
            "description": f"Auto-generated method for {method_name}"
        }
    
    # Render template
    template = Template(CLIENT_TEMPLATE)
    generated_code = template.render(methods=methods)
    
    # Write to file
    with open(output_file, 'w') as f:
        f.write(generated_code)
    
    print(f"✅ Generated client using templates: {output_file}")

def map_type(api_type: str) -> str:
    """Map API types to Python types"""
    type_mapping = {
        "string": "str",
        "integer": "int", 
        "boolean": "bool",
        "List[Post]": "List[Post]",
        "User": "User",
        "Post": "Post"
    }
    return type_mapping.get(api_type, "Any")
```

### Running the Template Generator

```python
# Example usage
schema = {
    "get_user": {
        "method": "GET",
        "params": {"user_id": "integer"},
        "returns": "User"
    },
    "create_post": {
        "method": "POST",
        "params": {"title": "string", "content": "string", "author_id": "integer"},
        "returns": "Post"
    }
}

generate_client_template(schema, "template_client.py")
```

### Generated Output

```python
# template_client.py (generated)
from typing import List
import requests

class APIClient:
    def __init__(self, base_url: str):
        self.base_url = base_url
        self.session = requests.Session()
    
    def get_user(self, user_id: int) -> User:
        """Auto-generated method for get_user"""
        response = self.session.get(
            f"{self.base_url}/get_user",
            params={
                "user_id": user_id,
            }
        )
        response.raise_for_status()
        return response.json()
    
    def create_post(self, title: str, content: str, author_id: int) -> Post:
        """Auto-generated method for create_post"""
        response = self.session.post(
            f"{self.base_url}/create_post",
            json={
                "title": title,
                "content": content,
                "author_id": author_id,
            }
        )
        response.raise_for_status()
        return response.json()
```

## Approach 2: AST-Based Generation

AST-based generation works at the syntax tree level, giving you precise control over the generated code structure.

### Implementation

```python
# ast_generator.py
import ast
from typing import Dict, Any

class ASTClientGenerator:
    """Generate API client using Python AST manipulation"""
    
    def __init__(self, schema: Dict[str, Any]):
        self.schema = schema
        self.module = ast.Module(body=[], type_ignores=[])
        
    def generate(self) -> str:
        """Generate the complete client module"""
        self._add_imports()
        self._add_client_class()
        return ast.unparse(self.module)
    
    def _add_imports(self):
        """Add necessary imports to the module"""
        imports = [
            ast.Import(names=[ast.alias(name='requests', asname=None)]),
            ast.ImportFrom(
                module='typing',
                names=[ast.alias(name='List', asname=None)],
                level=0
            )
        ]
        self.module.body.extend(imports)
    
    def _add_client_class(self):
        """Create the main APIClient class"""
        client_class = ast.ClassDef(
            name='APIClient',
            bases=[],
            keywords=[],
            body=[],
            decorator_list=[]
        )
        
        # Add __init__ method
        client_class.body.append(self._create_init_method())
        
        # Add API methods
        for method_name, method_info in self.schema.items():
            method_ast = self._create_api_method(method_name, method_info)
            client_class.body.append(method_ast)
        
        self.module.body.append(client_class)
    
    def _create_init_method(self) -> ast.FunctionDef:
        """Create the __init__ method"""
        return ast.FunctionDef(
            name='__init__',
            args=ast.arguments(
                args=[
                    ast.arg(arg='self', annotation=None),
                    ast.arg(arg='base_url', annotation=ast.Name(id='str', ctx=ast.Load()))
                ],
                defaults=[],
                kwonlyargs=[],
                kw_defaults=[],
                posonlyargs=[]
            ),
            body=[
                # self.base_url = base_url
                ast.Assign(
                    targets=[ast.Attribute(
                        value=ast.Name(id='self', ctx=ast.Load()),
                        attr='base_url',
                        ctx=ast.Store()
                    )],
                    value=ast.Name(id='base_url', ctx=ast.Load())
                ),
                # self.session = requests.Session()
                ast.Assign(
                    targets=[ast.Attribute(
                        value=ast.Name(id='self', ctx=ast.Load()),
                        attr='session',
                        ctx=ast.Store()
                    )],
                    value=ast.Call(
                        func=ast.Attribute(
                            value=ast.Name(id='requests', ctx=ast.Load()),
                            attr='Session',
                            ctx=ast.Load()
                        ),
                        args=[],
                        keywords=[]
                    )
                )
            ],
            decorator_list=[]
        )
    
    def _create_api_method(self, method_name: str, method_info: Dict[str, Any]) -> ast.FunctionDef:
        """Create an individual API method"""
        
        # Build method arguments
        args = [ast.arg(arg='self', annotation=None)]
        for param_name, param_type in method_info["params"].items():
            python_type = self._map_type_to_ast(param_type)
            args.append(ast.arg(arg=param_name, annotation=python_type))
        
        # Create return type annotation
        return_type = self._map_type_to_ast(method_info["returns"])
        
        # Build method body
        if method_info["method"] == "GET":
            body = self._create_get_request_body(method_name, method_info["params"])
        else:
            body = self._create_post_request_body(method_name, method_info["params"])
        
        return ast.FunctionDef(
            name=method_name,
            args=ast.arguments(
                args=args,
                defaults=[],
                kwonlyargs=[],
                kw_defaults=[],
                posonlyargs=[]
            ),
            body=body,
            decorator_list=[],
            returns=return_type
        )
    
    def _create_get_request_body(self, method_name: str, params: Dict[str, str]) -> list:
        """Create body for GET request method"""
        param_dict = ast.Dict(
            keys=[ast.Constant(value=param) for param in params.keys()],
            values=[ast.Name(id=param, ctx=ast.Load()) for param in params.keys()]
        )
        
        return [
            # response = self.session.get(f"{self.base_url}/method_name", params={...})
            ast.Assign(
                targets=[ast.Name(id='response', ctx=ast.Store())],
                value=ast.Call(
                    func=ast.Attribute(
                        value=ast.Attribute(
                            value=ast.Name(id='self', ctx=ast.Load()),
                            attr='session',
                            ctx=ast.Load()
                        ),
                        attr='get',
                        ctx=ast.Load()
                    ),
                    args=[
                        ast.JoinedStr(values=[
                            ast.Constant(value='{'),
                            ast.FormattedValue(
                                value=ast.Attribute(
                                    value=ast.Name(id='self', ctx=ast.Load()),
                                    attr='base_url',
                                    ctx=ast.Load()
                                ),
                                conversion=-1,
                                format_spec=None
                            ),
                            ast.Constant(value=f'}/{method_name}')
                        ])
                    ],
                    keywords=[
                        ast.keyword(arg='params', value=param_dict)
                    ]
                )
            ),
            # response.raise_for_status()
            ast.Expr(value=ast.Call(
                func=ast.Attribute(
                    value=ast.Name(id='response', ctx=ast.Load()),
                    attr='raise_for_status',
                    ctx=ast.Load()
                ),
                args=[],
                keywords=[]
            )),
            # return response.json()
            ast.Return(value=ast.Call(
                func=ast.Attribute(
                    value=ast.Name(id='response', ctx=ast.Load()),
                    attr='json',
                    ctx=ast.Load()
                ),
                args=[],
                keywords=[]
            ))
        ]
    
    def _create_post_request_body(self, method_name: str, params: Dict[str, str]) -> list:
        """Create body for POST request method"""
        param_dict = ast.Dict(
            keys=[ast.Constant(value=param) for param in params.keys()],
            values=[ast.Name(id=param, ctx=ast.Load()) for param in params.keys()]
        )
        
        return [
            # response = self.session.post(f"{self.base_url}/method_name", json={...})
            ast.Assign(
                targets=[ast.Name(id='response', ctx=ast.Store())],
                value=ast.Call(
                    func=ast.Attribute(
                        value=ast.Attribute(
                            value=ast.Name(id='self', ctx=ast.Load()),
                            attr='session',
                            ctx=ast.Load()
                        ),
                        attr='post',
                        ctx=ast.Load()
                    ),
                    args=[
                        ast.JoinedStr(values=[
                            ast.Constant(value='{'),
                            ast.FormattedValue(
                                value=ast.Attribute(
                                    value=ast.Name(id='self', ctx=ast.Load()),
                                    attr='base_url',
                                    ctx=ast.Load()
                                ),
                                conversion=-1,
                                format_spec=None
                            ),
                            ast.Constant(value=f'}/{method_name}')
                        ])
                    ],
                    keywords=[
                        ast.keyword(arg='json', value=param_dict)
                    ]
                )
            ),
            # response.raise_for_status()
            ast.Expr(value=ast.Call(
                func=ast.Attribute(
                    value=ast.Name(id='response', ctx=ast.Load()),
                    attr='raise_for_status',
                    ctx=ast.Load()
                ),
                args=[],
                keywords=[]
            )),
            # return response.json()
            ast.Return(value=ast.Call(
                func=ast.Attribute(
                    value=ast.Name(id='response', ctx=ast.Load()),
                    attr='json',
                    ctx=ast.Load()
                ),
                args=[],
                keywords=[]
            ))
        ]
    
    def _map_type_to_ast(self, type_name: str) -> ast.AST:
        """Convert string type to AST type annotation"""
        type_mapping = {
            "string": ast.Name(id='str', ctx=ast.Load()),
            "integer": ast.Name(id='int', ctx=ast.Load()),
            "boolean": ast.Name(id='bool', ctx=ast.Load()),
            "User": ast.Name(id='User', ctx=ast.Load()),
            "Post": ast.Name(id='Post', ctx=ast.Load()),
        }
        
        # Handle List types
        if type_name.startswith("List[") and type_name.endswith("]"):
            inner_type = type_name[5:-1]
            return ast.Subscript(
                value=ast.Name(id='List', ctx=ast.Load()),
                slice=self._map_type_to_ast(inner_type),
                ctx=ast.Load()
            )
        
        return type_mapping.get(type_name, ast.Name(id='Any', ctx=ast.Load()))

def generate_client_ast(schema: dict, output_file: str):
    """Generate client using AST manipulation"""
    generator = ASTClientGenerator(schema)
    generated_code = generator.generate()
    
    # Optional: Format with black
    try:
        import black
        generated_code = black.format_str(generated_code, mode=black.Mode())
    except ImportError:
        pass
    
    with open(output_file, 'w') as f:
        f.write(generated_code)
    
    print(f"✅ Generated client using AST: {output_file}")
```

### Running the AST Generator

```python
# Example usage
schema = {
    "get_user": {
        "method": "GET",
        "params": {"user_id": "integer"},
        "returns": "User"
    },
    "create_post": {
        "method": "POST",
        "params": {"title": "string", "content": "string", "author_id": "integer"},
        "returns": "Post"
    }
}

generate_client_ast(schema, "ast_client.py")
```

## Comparative Analysis: Templates vs AST

### 1. **Code Complexity and Readability**

**Templates (Jinja2):**
```python
# Template - easy to read, looks like target code
def {{ method_name }}(self, {% for param, type in params %}{{ param }}: {{ type }}{% endfor %}):
    return self.session.get(f"{self.base_url}/{{ method_name }}")
```

**AST:**
```python
# AST - more verbose but precise
ast.FunctionDef(
    name=method_name,
    args=ast.arguments(args=[...]),
    body=[ast.Return(value=ast.Call(...))]
)
```

**Winner: Templates** - Much easier to read and understand.

### 2. **Type Safety and Precision**

**Templates:** Generate text that might be syntactically invalid.
```python
# This template could generate invalid Python
def {{ name }}({{ "invalid syntax" if condition }}):
    pass
```

**AST:** Guarantees syntactically valid Python by construction.
```python
# AST ensures structural validity
ast.FunctionDef(name="valid_name", ...)  # Always creates valid syntax
```

**Winner: AST** - Structural guarantees prevent syntax errors.

### 3. **Complex Code Generation**

Consider generating a class with decorators and complex inheritance:

**Templates:** Become unwieldy with complex nested structures.
```jinja2
{% for decorator in decorators %}
@{{ decorator.name }}{% if decorator.args %}({{ decorator.args | join(", ") }}){% endif %}
{% endfor %}
class {{ class_name }}{% if base_classes %}({{ base_classes | join(", ") }}){% endif %}:
    {% for method in methods %}
        {% for method_decorator in method.decorators %}
    @{{ method_decorator }}
        {% endfor %}
    def {{ method.name }}(self, {{ method.params | join(", ") }}):
        {{ method.body | indent(8) }}
    {% endfor %}
```

**AST:** Handles complexity naturally through composition.
```python
class_def = ast.ClassDef(
    name=class_name,
    bases=[ast.Name(id=base, ctx=ast.Load()) for base in base_classes],
    decorator_list=[self._create_decorator(d) for d in decorators],
    body=[self._create_method(m) for m in methods]
)
```

**Winner: AST** - Better handling of nested and complex structures.

### 4. **Performance**

**Templates:** Fast rendering, minimal overhead.
**AST:** More overhead from object creation and traversal.

For typical code generation tasks, performance difference is negligible.

**Winner: Templates** (marginally) - Less computational overhead.

### 5. **Debugging Experience**

**Templates:** Errors often manifest as invalid generated Python code.
```python
# Generated code might have issues
def (self, : int):  # Missing method name - template error
    pass
```

**AST:** Errors occur during generation, not in generated code.
```python
# AST errors are caught at generation time
ast.FunctionDef(name="", ...)  # Clear error: missing name
```

**Winner: AST** - Fail-fast with clearer error messages.

### 6. **Maintainability**

**Templates:** Easy for designers/non-programmers to modify.
**AST:** Requires deep Python knowledge to maintain.

**Winner: Templates** - Lower barrier to contribution.

## Advanced Example: Dynamic Import Generation

Here's where the differences become stark. Suppose you need to generate dynamic imports based on runtime data:

### Templates Fall Short

```jinja2
{# This doesn't work - templates can't handle dynamic imports well #}
{% for module in dynamic_modules %}
try:
    from {{ module.path }} import {{ module.items | join(", ") }}
except ImportError:
    {{ module.fallback_code }}
{% endfor %}
```

Templates struggle because they can't easily:
1. Validate import paths
2. Handle complex fallback logic
3. Generate conditional imports based on Python version

### AST Excels

```python
def create_dynamic_imports(modules_config):
    """Generate complex import logic with fallbacks"""
    import_nodes = []
    
    for module_info in modules_config:
        # Create try-except block for each import
        try_body = [
            ast.ImportFrom(
                module=module_info["path"],
                names=[ast.alias(name=item, asname=None) for item in module_info["items"]],
                level=0
            )
        ]
        
        except_body = [
            ast.Expr(value=ast.Constant(value=f"Failed to import {module_info['path']}"))
        ]
        
        # Add version-specific handling
        if module_info.get("min_python_version"):
            version_check = create_version_check(module_info["min_python_version"])
            try_body = [ast.If(test=version_check, body=try_body, orelse=except_body)]
            except_body = []
        
        try_except = ast.Try(
            body=try_body,
            handlers=[
                ast.ExceptHandler(
                    type=ast.Name(id='ImportError', ctx=ast.Load()),
                    name=None,
                    body=except_body
                )
            ],
            orelse=[],
            finalbody=[]
        )
        
        import_nodes.append(try_except)
    
    return import_nodes
```

## Decision Framework: When to Use Which Approach

### Use Templates When:

1. **Simple transformations** - Converting straightforward data to code
2. **Human readability matters** - Templates will be modified by non-programmers
3. **Rapid prototyping** - Need to iterate quickly on code structure
4. **Output resembles input** - Template looks similar to final code

### Use AST When:

1. **Complex code structures** - Nested classes, decorators, intricate logic
2. **Type safety critical** - Generated code must be syntactically perfect
3. **Dynamic generation** - Code structure depends on runtime analysis
4. **Integration with Python tooling** - Need to analyze or transform generated code

### Hybrid Approach

For maximum flexibility, combine both:

```python
class HybridGenerator:
    def __init__(self):
        self.ast_generator = ASTClientGenerator()
        self.template_engine = Environment(loader=FileSystemLoader('.'))
    
    def generate_client(self, schema):
        if self._is_simple_schema(schema):
            # Use templates for simple cases
            return self._generate_with_template(schema)
        else:
            # Use AST for complex cases
            return self._generate_with_ast(schema)
    
    def _is_simple_schema(self, schema):
        """Determine if schema is simple enough for templates"""
        return all(
            len(method["params"]) <= 5 and 
            not any("List[" in param_type for param_type in method["params"].values())
            for method in schema.values()
        )
```

## Conclusion

Both template-based and AST-based code generation have their place in Python metaprogramming:

- **Templates excel** at simple, readable transformations where human maintainability matters
- **AST excels** at complex, precise code generation where structural guarantees are essential

The choice depends on your specific needs:
- For simple API clients, configuration files, or boilerplate reduction → **Templates**
- For complex SDKs, advanced metaprogramming, or tools that manipulate code → **AST**

The most powerful systems often use both approaches strategically, choosing the right tool for each specific generation task.

Remember: the goal isn't to use the most sophisticated approach, but to use the approach that best solves your problem while remaining maintainable for your team.