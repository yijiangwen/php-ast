php-ast
=======

This extension exposes the abstract syntax tree generated by PHP 7.

**This is the documentation for version 1.0.x. See also [documentation for version 0.1.x][v0_1_x].**

Table of Contents
-----------------

 * [Installation](#installation)
 * [API overview](#api-overview)
 * [Basic usage](#basic-usage)
 * [Example](#example)
 * [Metadata](#metadata)
 * [Flags](#flags)
 * [AST node kinds](#ast-node-kinds)
 * [AST versioning](#ast-versioning)
 * [Differences to PHP-Parser](#differences-to-php-parser)

Installation
------------

**Windows**: Download a [prebuilt Windows DLL](http://windows.php.net/downloads/pecl/releases/ast/)
and move it into the `ext/` directory of your PHP installation. Furthermore add
`extension=php_ast.dll` to your `php.ini` file.

**Unix (PECL)**: Run `pecl install ast` and add `extension=ast.so` to your `php.ini`.

**Unix (Compile)**: Compile and install the extension as follows.

```sh
phpize
./configure
make
sudo make install
```

Additionally add `extension=ast.so` to your `php.ini` file.

API overview
------------

Defines:

 * `ast\Node` class
 * `ast\Metadata` class
 * `ast\AST_*` kind constants
 * `ast\flags\*` flag constants
 * `ast\parse_file(string $filename, int $version)`
 * `ast\parse_code(string $code, int $version [, string $filename = "string code"])`
 * `ast\get_kind_name(int $kind)`
 * `ast\kind_uses_flags(int $kind)`
 * `ast\get_metadata()`
 * `ast\get_supported_versions(bool $exclude_deprecated = false)`

Basic usage
-----------

Code can be parsed using either `ast\parse_code()`, which accepts a code string, or
`ast\parse_file()`, which accepts a file path. Additionally, both functions require a `$version`
argument to ensure forward-compatibility. The current version is 70.

```php
$ast = ast\parse_code('<?php ...', $version=70);
// or
$ast = ast\parse_file('file.php', $version=70);
```

The abstract syntax tree returned by these functions consists of `ast\Node` objects.
`ast\Node` is declared as follows:

```php
namespace ast;
class Node {
    public $kind;
    public $flags;
    public $lineno;
    public $children;
}
```

The `kind` property specifies the type of the node. It is an integral value, which corresponds to
one of the `ast\AST_*` constants, for example `ast\AST_STMT_LIST`. See the
[AST node kinds section](#ast-node-kinds) for an overview of the available node kinds.

The `flags` property contains node specific flags. It is always defined, but for most nodes it is
always zero. See the [flags section](#flags) for a list of flags supported by the different node
kinds.

The `lineno` property specifies the *starting* line number of the node.

The `children` property contains an array of child-nodes. These children can be either other
`ast\Node` objects or plain values. There are two general categories of nodes: Normal AST nodes,
which have a fixed set of named child nodes, as well as list nodes, which have a variable number
of children. The [AST node kinds section](#ast-node-kinds) contains a list of the child names for
the different node kinds.

Example
-------

Simple usage example:

```php
<?php

$code = <<<'EOC'
<?php
$var = 42;
EOC;

var_dump(ast\parse_code($code, $version=70));

// Output:
object(ast\Node)#1 (4) {
  ["kind"]=>
  int(133)
  ["flags"]=>
  int(0)
  ["lineno"]=>
  int(1)
  ["children"]=>
  array(1) {
    [0]=>
    object(ast\Node)#2 (4) {
      ["kind"]=>
      int(517)
      ["flags"]=>
      int(0)
      ["lineno"]=>
      int(2)
      ["children"]=>
      array(2) {
        ["var"]=>
        object(ast\Node)#3 (4) {
          ["kind"]=>
          int(256)
          ["flags"]=>
          int(0)
          ["lineno"]=>
          int(2)
          ["children"]=>
          array(1) {
            ["name"]=>
            string(3) "var"
          }
        }
        ["expr"]=>
        int(42)
      }
    }
  }
}
```

The [`util.php`][util] file defines an `ast_dump()` function, which can be used to create a more
compact and human-readable dump of the AST structure:

```php
<?php

require 'path/to/util.php';

$code = <<<'EOC'
<?php
$var = 42;
EOC;

echo ast_dump(ast\parse_code($code, $version=70)), "\n";

// Output:
AST_STMT_LIST
    0: AST_ASSIGN
        var: AST_VAR
            name: "var"
        expr: 42
```

To additionally show line numbers pass the `AST_DUMP_LINENOS` option as the second argument to
`ast_dump()`.

A more substantial AST dump can be found [in the tests][test_dump].

Metadata
--------

There are a number of functions which provide meta-information for the AST structure:

`ast\get_kind_name()` returns a string name for an integral node kind.

`ast\kind_uses_flags()` determines whether the `flags` of a node kind may ever be non-zero.

`ast\get_metadata()` returns metadata about all AST node kinds. The return value is a map from AST
node kinds to `ast\Metadata` objects defined as follows.

```php
namespace ast;
class Metadata
{
    public $kind;
    public $name;
    public $flags;
    public $flagsCombinable;
}
```

The `kind` is the integral node kind, while `name` is the same name as returned by the
`get_kind_name()` function.

`flags` is an array of flag constant names, which may be used by the node kind. `flagsCombinable`
specifies whether the flag should be checked using `===` (not combinable) or using `&` (combinable).
Together these two values provide programmatic access to the information listed in the
[flags section](#flags).

The AST metadata is intended for use in tooling for working the AST, such as AST dumpers.

Flags
-----

This section lists which flags are used by which AST node kinds. The "combinable" flags can be
combined using bitwise or and should be checked by using `$ast->flags & ast\flags\FOO`. The
"exclusive" flags are used standalone and should be checked using `$ast->flags === ast\flags\BAR`.

```
// Used by ast\AST_NAME (exclusive)
ast\flags\NAME_FQ (= 0)    // example: \Foo\Bar
ast\flags\NAME_NOT_FQ      // example: Foo\Bar
ast\flags\NAME_RELATIVE    // example: namespace\Foo\Bar

// Used by ast\AST_METHOD, ast\AST_PROP_DECL, ast\AST_CLASS_CONST_DECL,
// ast\AST_PROP_GROUP and ast\AST_TRAIT_ALIAS (combinable)
ast\flags\MODIFIER_PUBLIC
ast\flags\MODIFIER_PROTECTED
ast\flags\MODIFIER_PRIVATE
ast\flags\MODIFIER_STATIC
ast\flags\MODIFIER_ABSTRACT
ast\flags\MODIFIER_FINAL

// Used by ast\AST_CLOSURE, ast\AST_ARROW_FUNC (combinable)
ast\flags\MODIFIER_STATIC

// Used by ast\AST_FUNC_DECL, ast\AST_METHOD, ast\AST_CLOSURE, ast\AST_ARROW_FUNC (combinable)
ast\flags\FUNC_RETURNS_REF  // legacy alias: ast\flags\RETURNS_REF
ast\flags\FUNC_GENERATOR    // used only in PHP >= 7.1

// Used by ast\AST_CLOSURE_VAR
ast\flags\CLOSURE_USE_REF

// Used by ast\AST_CLASS (exclusive)
ast\flags\CLASS_ABSTRACT
ast\flags\CLASS_FINAL
ast\flags\CLASS_TRAIT
ast\flags\CLASS_INTERFACE
ast\flags\CLASS_ANONYMOUS

// Used by ast\AST_PARAM (combinable)
ast\flags\PARAM_REF
ast\flags\PARAM_VARIADIC
ast\flags\MODIFIER_PUBLIC
ast\flags\MODIFIER_PROTECTED
ast\flags\MODIFIER_PRIVATE

// Used by ast\AST_TYPE (exclusive)
ast\flags\TYPE_ARRAY
ast\flags\TYPE_CALLABLE
ast\flags\TYPE_VOID
ast\flags\TYPE_BOOL
ast\flags\TYPE_LONG
ast\flags\TYPE_DOUBLE
ast\flags\TYPE_STRING
ast\flags\TYPE_ITERABLE
ast\flags\TYPE_OBJECT
ast\flags\TYPE_NULL    // php 8.0 union types
ast\flags\TYPE_FALSE   // php 8.0 union types
ast\flags\TYPE_STATIC  // php 8.0 static return type
ast\flags\TYPE_MIXED   // php 8.0 mixed type

// Used by ast\AST_CAST (exclusive)
ast\flags\TYPE_NULL
ast\flags\TYPE_BOOL
ast\flags\TYPE_LONG
ast\flags\TYPE_DOUBLE
ast\flags\TYPE_STRING
ast\flags\TYPE_ARRAY
ast\flags\TYPE_OBJECT

// Used by ast\AST_UNARY_OP (exclusive)
ast\flags\UNARY_BOOL_NOT
ast\flags\UNARY_BITWISE_NOT
ast\flags\UNARY_MINUS
ast\flags\UNARY_PLUS
ast\flags\UNARY_SILENCE

// Used by ast\AST_BINARY_OP and ast\AST_ASSIGN_OP (exclusive)
ast\flags\BINARY_BITWISE_OR
ast\flags\BINARY_BITWISE_AND
ast\flags\BINARY_BITWISE_XOR
ast\flags\BINARY_CONCAT
ast\flags\BINARY_ADD
ast\flags\BINARY_SUB
ast\flags\BINARY_MUL
ast\flags\BINARY_DIV
ast\flags\BINARY_MOD
ast\flags\BINARY_POW
ast\flags\BINARY_SHIFT_LEFT
ast\flags\BINARY_SHIFT_RIGHT
ast\flags\BINARY_COALESCE

// Used by ast\AST_BINARY_OP (exclusive)
ast\flags\BINARY_BOOL_AND
ast\flags\BINARY_BOOL_OR
ast\flags\BINARY_BOOL_XOR
ast\flags\BINARY_IS_IDENTICAL
ast\flags\BINARY_IS_NOT_IDENTICAL
ast\flags\BINARY_IS_EQUAL
ast\flags\BINARY_IS_NOT_EQUAL
ast\flags\BINARY_IS_SMALLER
ast\flags\BINARY_IS_SMALLER_OR_EQUAL
ast\flags\BINARY_IS_GREATER
ast\flags\BINARY_IS_GREATER_OR_EQUAL
ast\flags\BINARY_SPACESHIP

// Used by ast\AST_MAGIC_CONST (exclusive)
ast\flags\MAGIC_LINE
ast\flags\MAGIC_FILE
ast\flags\MAGIC_DIR
ast\flags\MAGIC_NAMESPACE
ast\flags\MAGIC_FUNCTION
ast\flags\MAGIC_METHOD
ast\flags\MAGIC_CLASS
ast\flags\MAGIC_TRAIT

// Used by ast\AST_USE, ast\AST_GROUP_USE and ast\AST_USE_ELEM (exclusive)
ast\flags\USE_NORMAL
ast\flags\USE_FUNCTION
ast\flags\USE_CONST

// Used by ast\AST_INCLUDE_OR_EVAL (exclusive)
ast\flags\EXEC_EVAL
ast\flags\EXEC_INCLUDE
ast\flags\EXEC_INCLUDE_ONCE
ast\flags\EXEC_REQUIRE
ast\flags\EXEC_REQUIRE_ONCE

// Used by ast\AST_ARRAY (exclusive), since PHP 7.1
ast\flags\ARRAY_SYNTAX_SHORT
ast\flags\ARRAY_SYNTAX_LONG
ast\flags\ARRAY_SYNTAX_LIST

// Used by ast\AST_ARRAY_ELEM (exclusive)
ast\flags\ARRAY_ELEM_REF

// Used by ast\AST_DIM (combinable), since PHP 7.4
ast\flags\DIM_ALTERNATIVE_SYNTAX

// Used by ast\AST_CONDITIONAL (combinable), since PHP 7.4
ast\flags\PARENTHESIZED_CONDITIONAL
```

AST node kinds
--------------

This section lists the AST node kinds that are supported and the names of their child nodes.

```
AST_ARRAY_ELEM:       value, key
AST_ARROW_FUNC:       name, docComment, params, stmts, returnType, attributes
AST_ASSIGN:           var, expr
AST_ASSIGN_OP:        var, expr
AST_ASSIGN_REF:       var, expr
AST_ATTRIBUTE:        class, args            // php 8.0+ attributes
AST_BINARY_OP:        left, right
AST_BREAK:            depth
AST_CALL:             expr, args
AST_CAST:             expr
AST_CATCH:            class, var, stmts
AST_CLASS:            name, docComment, extends, implements, stmts
AST_CLASS_CONST:      class, const
AST_CLASS_CONST_GROUP class, attributes      // version 80+
AST_CLASS_NAME:       class                  // version 70+
AST_CLONE:            expr
AST_CLOSURE:          name, docComment, params, uses, stmts, returnType, attributes
AST_CLOSURE_VAR:      name
AST_CONDITIONAL:      cond, true, false
AST_CONST:            name
AST_CONST_ELEM:       name, value, docComment
AST_CONTINUE:         depth
AST_DECLARE:          declares, stmts
AST_DIM:              expr, dim
AST_DO_WHILE:         stmts, cond
AST_ECHO:             expr
AST_EMPTY:            expr
AST_EXIT:             expr
AST_FOR:              init, cond, loop, stmts
AST_FOREACH:          expr, value, key, stmts
AST_FUNC_DECL:        name, docComment, params, stmts, returnType, attributes
                      uses                   // prior to version 60
AST_GLOBAL:           var
AST_GOTO:             label
AST_GROUP_USE:        prefix, uses
AST_HALT_COMPILER:    offset
AST_IF_ELEM:          cond, stmts
AST_INCLUDE_OR_EVAL:  expr
AST_INSTANCEOF:       expr, class
AST_ISSET:            var
AST_LABEL:            name
AST_MAGIC_CONST:
AST_METHOD:           name, docComment, params, stmts, returnType, attributes
                      uses                   // prior to version 60
AST_METHOD_CALL:      expr, method, args
AST_METHOD_REFERENCE: class, method
AST_NAME:             name
AST_NAMESPACE:        name, stmts
AST_NEW:              class, args
AST_NULLABLE_TYPE:    type                   // Used only since PHP 7.1
AST_PARAM:            type, name, default, attributes, docComment
AST_POST_DEC:         var
AST_POST_INC:         var
AST_PRE_DEC:          var
AST_PRE_INC:          var
AST_PRINT:            expr
AST_PROP:             expr, prop
AST_PROP_ELEM:        name, default, docComment
AST_PROP_GROUP:       type, props, attributes // version 70+
AST_REF:              var                    // only used in foreach ($a as &$v)
AST_RETURN:           expr
AST_SHELL_EXEC:       expr
AST_STATIC:           var, default
AST_STATIC_CALL:      class, method, args
AST_STATIC_PROP:      class, prop
AST_SWITCH:           cond, stmts
AST_SWITCH_CASE:      cond, stmts
AST_THROW:            expr
AST_TRAIT_ALIAS:      method, alias
AST_TRAIT_PRECEDENCE: method, insteadof
AST_TRY:              try, catches, finally
AST_TYPE:
AST_UNARY_OP:         expr
AST_UNPACK:           expr
AST_UNSET:            var
AST_USE_ELEM:         name, alias
AST_USE_TRAIT:        traits, adaptations
AST_VAR:              name
AST_WHILE:            cond, stmts
AST_YIELD:            value, key
AST_YIELD_FROM:       expr

// List nodes (numerically indexed children):
AST_ARG_LIST
AST_ARRAY
AST_ATTRIBUTE_LIST        // php 8.0+ attributes
AST_CATCH_LIST
AST_CLASS_CONST_DECL
AST_CLOSURE_USES
AST_CONST_DECL
AST_ENCAPS_LIST           // interpolated string: "foo$bar"
AST_EXPR_LIST
AST_IF
AST_LIST
AST_NAME_LIST
AST_PARAM_LIST
AST_PROP_DECL
AST_STMT_LIST
AST_SWITCH_LIST
AST_TRAIT_ADAPTATIONS
AST_USE
AST_TYPE_UNION            // php 8.0+ union types
```

AST Versioning
--------------

The `ast\parse_code()` and `ast\parse_file()` functions each accept a required AST `$version`
argument. The idea behind the AST version is that we may need to change the format of the AST to
account for new features in newer PHP versions, or to improve it in other ways. Such changes will
always be made under a new AST version, while previous formats continue to be supported for some
time.

Old AST versions may be deprecated in minor versions and removed in major versions of the AST extension.

A list of currently supported versions is available through `ast\get_supported_versions()`. The
function accepts a boolean argument that determines whether deprecated versions should be excluded.

In the following the changes in the respective AST versions, as well as their current support state,
are listed.

### 80 (experimental)

Available since 1.0.7 (XXX).

* `mixed` type hints are now reported as an `AST_TYPE` with type `TYPE_MIXED` instead of an `AST_NAME`.
* `AST_CLASS_CONST_GROUP` nodes are emitted for php 8.0 class constant declarations with a `const` and an optional `attributes` node.

### 70 (current)

Supported since 1.0.1 (2019-02-11).

* `AST_PROP_GROUP` was added to support PHP 7.4's typed properties.
  The property visibility modifiers are now part of `AST_PROP_GROUP` instead of `AST_PROP_DECL`.

  Note that property group type information is only available with AST versions 70+.
* `AST_CLASS_NAME` is created instead of `AST_CLASS_CONST` for `SomeClass::class`.

### 60 (stable)

Supported since 0.1.7 (2018-10-06).

* `AST_FUNC_DECL` and `AST_METHOD` no longer generate a `uses` child. Previously this child was
  always `null`.
* `AST_CONST_ELEM` now always has a `docComment` child. Previously it was absent on PHP 7.0.
  On PHP 7.0 the value is always `null`.

### 50 (stable)

Supported since 0.1.5 (2017-07-19).

This is the oldest AST version available in 1.0.0. See the
[0.1.x AST version changelog][v0_1_x_versions] for information on changes prior to this version.

Differences to PHP-Parser
-------------------------

Next to php-ast I also maintain the [PHP-Parser][php-parser] library, which has some overlap with
this extension. This section summarizes the main differences between php-ast and PHP-Parser so you
may decide which is preferable for your use-case.

The primary difference is that php-ast is a PHP extension (written in C) which exports the AST
internally used by PHP 7. PHP-Parser on the other hand is library written in PHP. This has a number
of consequences:

 * php-ast is significantly faster than PHP-Parser, because the AST construction is implemented in
   C.
 * php-ast needs to be installed as an extension, on Linux either by compiling it manually or
   retrieving it from a package manager, on Windows by loading a DLL. PHP-Parser is installed as a
   Composer dependency.
 * php-ast only runs on PHP >= 7.0, as prior versions did not use an internal AST. PHP-Parser
   supports PHP >= 5.5.
 * php-ast may only parse code that is syntactically valid on the version of PHP it runs on. This
   means that it's not possible to parse code using features of newer versions (e.g. PHP 7.1 code
   while running on PHP 7.0). Similarly, it is not possible to parse code that is no longer
   syntactically valid on the used version (e.g. some PHP 5 code may no longer be parsed -- however
   most code will work). PHP-Parser supports parsing both newer and older (up to PHP 5.2) versions.
 * php-ast can only parse code which is syntactically valid, while PHP-Parser can provide a partial
   AST for code that contains errors (e.g., because it is currently being edited).
 * php-ast only provides the starting line number (and for declarations the ending line number) of
   nodes, because this is the only part that PHP itself stores. PHP-Parser provides precise file
   offsets.

There are a number of differences in the AST representation and available support code:

 * The PHP-Parser library uses a separate class for every node type, with child nodes stored as
   type-annotated properties. php-ast uses one class for everything, with children stored as
   arrays. The former approach is friendlier to developers because it has very good IDE integration.
   The php-ast extension does not use separate classes, because registering hundreds of classes was
   judged a no-go for a bundled extension.
 * The PHP-Parser library contains various support code for working with the AST, while php-ast only
   handles AST construction. The reason for this is that implementing this support code in C is
   extremely complicated and there is little other benefit to implementing it in C. The main
   components that PHP-Parser offers that may be of interest are:
    * Node dumper (human readable representation): While the php-ast extension does not directly
      implement this, a `ast_dump` function is provided in the `util.php` file.
    * Pretty printer (converting the AST back to PHP code): This is not provided natively, but the
      [php-ast-reverter][php-ast-reverter] package implements this functionality.
    * Name resolution (resolving namespace prefixes and aliases): There is currently no standalone
      package for this.
    * AST traversal / visitation: There is currently no standalone package for this either, but
      implementing a recursive AST walk is easy.

The [tolerant-php-parser-to-php-ast][tolerant-php-parser-to-php-ast] project can convert the AST produced by
[tolerant-php-parser][tolerant-php-parser] (Another pure PHP parser library) into the format used by the php-ast extension.
This can be used as a slow fallback in
case the php-ast extension is not available. It may also be used to produce a partial php-ast output
for code with syntax errors.

  [parser]: http://lxr.php.net/xref/PHP_TRUNK/Zend/zend_language_parser.y
  [util]: https://github.com/nikic/php-ast/blob/master/util.php
  [test_dump]: https://github.com/nikic/php-ast/blob/master/tests/001.phpt
  [php-parser]: https://github.com/nikic/PHP-Parser
  [php-ast-reverter]: https://github.com/tpunt/php-ast-reverter
  [tolerant-php-parser]: https://github.com/Microsoft/tolerant-php-parser
  [tolerant-php-parser-to-php-ast]: https://github.com/tysonandre/tolerant-php-parser-to-php-ast
  [v0_1_x]: https://github.com/nikic/php-ast/tree/v0.1.x#php-ast
  [v0_1_x_versions]: https://github.com/nikic/php-ast/tree/v0.1.x#ast-versioning
