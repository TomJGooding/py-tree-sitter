# Python Tree-sitter

[![CI][ci]](https://github.com/tree-sitter/py-tree-sitter/actions/workflows/ci.yml)
[![pypi][pypi]](https://pypi.org/project/tree-sitter/)
[![docs][docs]](https://tree-sitter.github.io/py-tree-sitter/)

This module provides Python bindings to the [tree-sitter] parsing library.

## Installation

The package has no library dependencies and provides pre-compiled wheels for all major platforms.

> [!NOTE]
> If your platform is not currently supported, please submit an [issue] on GitHub.

```sh
pip install tree-sitter
```

## Usage

### Setup

#### Install languages

Tree-sitter language implementations also provide pre-compiled binary wheels.
Let's take [Python][tree-sitter-python] as an example.

```sh
pip install tree-sitter-python
```

Then, you can load it as a `Language` object:

```python
import tree_sitter_python as tspython
from tree_sitter import Language, Parser

PY_LANGUAGE = Language(tspython.language())
```

### Basic parsing

Create a `Parser` and configure it to use a language:

```python
parser = Parser(PY_LANGUAGE)
```

Parse some source code:

```python
tree = parser.parse(
    bytes(
        """
def foo():
    if bar:
        baz()
""",
        "utf8"
    )
)
```

If you have your source code in some data structure other than a bytes object,
you can pass a "read" callable to the parse function.

The read callable can use either the byte offset or point tuple to read from
buffer and return source code as bytes object. An empty bytes object or None
terminates parsing for that line. The bytes must be encoded as UTF-8 or UTF-16.

For example, to use the byte offset with UTF-8 encoding:

```python
src = bytes(
    """
def foo():
    if bar:
        baz()
""",
    "utf8",
)


def read_callable_byte_offset(byte_offset, point):
    return src[byte_offset : byte_offset + 1]


tree = parser.parse(read_callable_byte_offset, encoding="utf8")
```

And to use the point:

```python
src_lines = ["\n", "def foo():\n", "    if bar:\n", "        baz()\n"]


def read_callable_point(byte_offset, point):
    row, column = point
    if row >= len(src_lines) or column >= len(src_lines[row]):
        return None
    return src_lines[row][column:].encode("utf8")


tree = parser.parse(read_callable_point, encoding="utf8")
```

Inspect the resulting `Tree`:

```python
root_node = tree.root_node
assert root_node.type == 'module'
assert root_node.start_point == (1, 0)
assert root_node.end_point == (4, 0)

function_node = root_node.children[0]
assert function_node.type == 'function_definition'
assert function_node.child_by_field_name('name').type == 'identifier'

function_name_node = function_node.children[1]
assert function_name_node.type == 'identifier'
assert function_name_node.start_point == (1, 4)
assert function_name_node.end_point == (1, 7)

function_body_node = function_node.child_by_field_name("body")

if_statement_node = function_body_node.child(0)
assert if_statement_node.type == "if_statement"

function_call_node = if_statement_node.child_by_field_name("consequence").child(0).child(0)
assert function_call_node.type == "call"

function_call_name_node = function_call_node.child_by_field_name("function")
assert function_call_name_node.type == "identifier"

function_call_args_node = function_call_node.child_by_field_name("arguments")
assert function_call_args_node.type == "argument_list"


assert str(root_node) == (
    "(module "
        "(function_definition "
            "name: (identifier) "
            "parameters: (parameters) "
            "body: (block "
                "(if_statement "
                    "condition: (identifier) "
                    "consequence: (block "
                        "(expression_statement (call "
                            "function: (identifier) "
                            "arguments: (argument_list))))))))"
)
```

Or, to use the byte offset with UTF-16 encoding:

```python
parser.language = JAVASCRIPT
source_code = bytes("'😎' && '🐍'", "utf16")

def read(byte_position, _):
    return source_code[byte_position: byte_position + 2]

tree = parser.parse(read, encoding="utf16")
root_node = tree.root_node
statement_node = root_node.children[0]
binary_node = statement_node.children[0]
snake_node = binary_node.children[2]
snake = source_code[snake_node.start_byte:snake_node.end_byte]

assert binary_node.type == "binary_expression"
assert snake_node.type == "string"
assert snake.decode("utf16") == "'🐍'"
```

### Walking syntax trees

If you need to traverse a large number of nodes efficiently, you can use
a `TreeCursor`:

```python
cursor = tree.walk()

assert cursor.node.type == "module"

assert cursor.goto_first_child()
assert cursor.node.type == "function_definition"

assert cursor.goto_first_child()
assert cursor.node.type == "def"

# Returns `False` because the `def` node has no children
assert not cursor.goto_first_child()

assert cursor.goto_next_sibling()
assert cursor.node.type == "identifier"

assert cursor.goto_next_sibling()
assert cursor.node.type == "parameters"

assert cursor.goto_parent()
assert cursor.node.type == "function_definition"
```

> [!IMPORTANT]
> Keep in mind that the cursor can only walk into children of the node that it started from.

See [examples/walk_tree.py] for a complete example of iterating over every node in a tree.

### Editing

When a source file is edited, you can edit the syntax tree to keep it in sync with
the source:

```python
new_src = src[:5] + src[5 : 5 + 2].upper() + src[5 + 2 :]

tree.edit(
    start_byte=5,
    old_end_byte=5,
    new_end_byte=5 + 2,
    start_point=(0, 5),
    old_end_point=(0, 5),
    new_end_point=(0, 5 + 2),
)
```

Then, when you're ready to incorporate the changes into a new syntax tree,
you can call `Parser.parse` again, but pass in the old tree:

```python
new_tree = parser.parse(new_src, tree)
```

This will run much faster than if you were parsing from scratch.

The `Tree.changed_ranges` method can be called on the _old_ tree to return
the list of ranges whose syntactic structure has been changed:

```python
for changed_range in tree.changed_ranges(new_tree):
    print("Changed range:")
    print(f"  Start point {changed_range.start_point}")
    print(f"  Start byte {changed_range.start_byte}")
    print(f"  End point {changed_range.end_point}")
    print(f"  End byte {changed_range.end_byte}")
```

### Pattern-matching

You can search for patterns in a syntax tree using a [tree query]:

```python
query = Query(
    PY_LANGUAGE,
    """
(function_definition
  name: (identifier) @function.def
  body: (block) @function.block)

(call
  function: (identifier) @function.call
  arguments: (argument_list) @function.args)
""",
)
```

#### Captures

```python
query_cursor = QueryCursor(query)
captures = query_cursor.captures(tree.root_node)
assert len(captures) == 4
assert captures["function.def"][0] == function_name_node
assert captures["function.block"][0] == function_body_node
assert captures["function.call"][0] == function_call_name_node
assert captures["function.args"][0] == function_call_args_node
```

#### Matches

```python
matches = query_cursor.matches(tree.root_node)
assert len(matches) == 2

# first match
assert matches[0][1]["function.def"] == [function_name_node]
assert matches[0][1]["function.block"] == [function_body_node]

# second match
assert matches[1][1]["function.call"] == [function_call_name_node]
assert matches[1][1]["function.args"] == [function_call_args_node]
```

The difference between the two methods is that `QueryCursor.matches()` groups captures into matches,
which is much more useful when your captures within a query relate to each other.

To try out and explore the code referenced in this README, check out [examples/usage.py].

[tree-sitter]: https://tree-sitter.github.io/tree-sitter/
[issue]: https://github.com/tree-sitter/py-tree-sitter/issues/new
[tree-sitter-python]: https://github.com/tree-sitter/tree-sitter-python
[tree query]: https://tree-sitter.github.io/tree-sitter/using-parsers/queries
[ci]: https://img.shields.io/github/actions/workflow/status/tree-sitter/py-tree-sitter/ci.yml?logo=github&label=CI
[pypi]: https://img.shields.io/pypi/v/tree-sitter?logo=pypi&logoColor=ffd242&label=PyPI
[docs]: https://img.shields.io/github/deployments/tree-sitter/py-tree-sitter/github-pages?logo=sphinx&label=Docs
[examples/walk_tree.py]: https://github.com/tree-sitter/py-tree-sitter/blob/master/examples/walk_tree.py
[examples/usage.py]: https://github.com/tree-sitter/py-tree-sitter/blob/master/examples/usage.py
