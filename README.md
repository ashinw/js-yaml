JS-YAML - YAML 1.2 parser and serializer for JavaScript
=======================================================

[![Build Status](https://secure.travis-ci.org/nodeca/js-yaml.png)](http://travis-ci.org/nodeca/js-yaml)

[Online Demo](http://nodeca.github.com/js-yaml/)


This is an implementation of [YAML](http://yaml.org/), a human friendly data
serialization language.

Previously, it was a native port of [PyYAML](http://pyyaml.org/). Now, it is
rewritten from scratch, directly from version 1.2 of the specification.


## Breaking changes in 1.x.x -> 2.0.x

If your code does not use neither custom tags nor explicitly specified Loader
class (the last argument of `load` function), **you are not required to change
anything**.

Otherwise, you should rewrite your tag constructors and custom Loader classes to
conform the new schema-based API. It consists of two classes: Schema and Type.
The both are described below in the API listing. For example of this, see
`examples/custom_type_old_1xx.js` and `examples/custom_type_new_20x.js` files.


### Summary

- The last argument of loader functions (`load`, `loadAll`, `safeLoad`, and
  `safeLoadAll`) was changed. Now, it is an `options` plain object. See the API
  listing on `load` function for details.
- `scan`, `parse`, `compose`, `addConstructor` functions and all of the classes
  like `Loader`, `Constructor`, `Resolver` was dropped because of complete
  architecture overhaul.


## Installation

### YAML module for node.js

```
npm install js-yaml
```


### CLI executable

If you want to inspect your YAML files from CLI, install js-yaml globally:

```
npm install js-yaml -g
```

#### Usage

```
usage: js-yaml [-h] [-v] [-c] [-j] [-t] file

Positional arguments:
  file           File with YAML document(s)

Optional arguments:
  -h, --help     Show this help message and exit.
  -v, --version  Show program's version number and exit.
  -c, --compact  Display errors in compact mode
  -j, --to-json  Output a non-funky boring JSON
  -t, --trace    Show stack trace on error
```


### Bundled YAML library for browsers

``` html
<script src="js-yaml.min.js"></script>
<script type="text/javascript">
var doc = jsyaml.load('greeting: hello\nname: world');
</script>
```

Browser support is still buggy, and mostly done to run online demo. If you
can help to improve browser compatibility and AMD support - rise pull request.

**Support of oldIEs** and some other prehistoric browsers is possible using
[es5-shims](https://github.com/kriskowal/es5-shim). Just include shims before
jsyaml to use it with outdated browsers.


## API

JS-YAML automatically registers handlers for `.yml` and `.yaml` files. You can
load them just with `require`. That's mostly equivalent to calling `load()` on
fetched content of a file. Just with one string!

``` javascript
require('js-yaml');

// Get document, or throw exception on error
var doc = require('/home/ixti/example.yml');

console.log(doc);
```

See `examples/` directory for the examples. Take especially a look at
`examples/loader.js` and `examples/dumper.js` files. 


### load (string [ , options ])

Parses `string` as single YAML document. Returns a JavaScript object or throws
`YAMLException` on error.

NOTE: This function **does not** understands multi-document sources, it throws
exception on those.

`options` is an optional hash-like object allows to change the loader's
behaviour. It may contain the following keys:

- `schema` specifies a schema to use. It's `DEFAULT_SCHEMA` by default. See
  below for more information about the schemas.
- `validate` (default true) enables/disables validation of the input stream
  according to YAML rules. If you are sure about the input, you can set it to
  false and (maybe) gain some additional performance.
- `strict` (default false) makes the loader to throw errors instead of warnings.
- `legacy` (default false) makes the loader to expect YAML 1.1 documents if such
  documents have no explicit %YAML directive.
- `name` (default null) is a string to be used as a file path in error/warning
  messages.
- `resolve` (default true) enables/disables resolving of nodes with non-specific
  `?` tag. That are plain scalars without any explicit tag. The result of
  switching it to false is that all of the plain scalars will loaded as strings.
  This may significantly increase parsing performance.


### loadAll (string, iterator [ , options ])

Same as `load()`, but understands multi-document sources and apply `iterator` to
each document.

``` javascript
var yaml = require('js-yaml');

yaml.loadAll(data, function (doc) {
  console.log(doc);
});
```


### safeLoad (string [ , options ])

Same as `load()` but uses SAFE_SCHEMA by default - only recommended tags of YAML
specification (no JavaScript-specific tags, e.g. `!!js/regexp`).


### safeLoadAll (string, iterator [ , options ])

Same as `loadAll()` but uses SAFE_SCHEMA by default - only recommended tags of
YAML specification (no JavaScript-specific tags, e.g. `!!js/regexp`).


### dump (object [ , options ])

Serializes `object` as single, bare YAML document. `options` is an optional
hash-like object allows to change the dumper's behaviour. It may contain the
following keys:

- `schema` specifies a schema to use. It's `DEFAULT_SCHEMA` by default. See
  below for more information about the schemas.
- `indent` (default 2) indentation width to use (in spaces).
- `flowLevel` (default -1) specifies level of nesting on which the dumper must
  switch to the flow style (i.e. JSON-like) of collection nodes. Use a nagitive
  number to disable.
- `styles` is a "tag" => "style" map. Each tag may have own set of styles. See
  below for full listing of standard tag styles.


### safeDump (object [ , options ])

Same as `dump()` but uses SAFE_SCHEMA by default - only recommended tags of YAML
specification (no JavaScript-specific tags, e.g. `!!js/regexp`).


### new Schema ({ include, implicit, explicit })

Constructs an object to use by the loader via the `schema` setting described
above. Schemas are collections of YAML type objects collected in `implicit`
and `explicit` arrays. The loader will try to resolve each plain scalar in a
document using the resolver function associeted with each type in the implicit
list. If a node has an explicit tag, the loader will look for it in the both
lists. `include` is an array of super schemas. When compiling a schema, the
loader will take types in bottom-top order; the specified schema comes first,
and all of super schemas come next in order of they are placed in the include
list. Recursively.

NOTE: Schemas must be immutable. So, it's better to use CONSTANT_LIKE_NAMES for
them.

There are predifined schemas in JS-YAML: `MINIMAL_SCHEMA`, `SAFE_SCHEMA`, and
`DEFAULT_SCHEMA`.


### new Type (tag, { loader, dumper })

Constructs a YAML type definition object. Such objects are used for validation,
resolving, interpreting, and representing of primitive YAML nodes: scalars
(strings), sequences (arrays), and mappings (objects). The second argument is an
object of two keys: `loader` and `dumper`. At least one of these must be
specified. Both of the keys are objects too.

**loader**
- `kind` (required) is a string identifier ("string", "array", or "object")
  restricts type of acceptable nodes.
- `resolver` (optional) is a function of two arguments: `object` is a primitive
  YAML node to resolve and `explicit` is a boolean value. When a type is
  contained in the implicit list of a schema, and a node has no explicit tag on
  it, `explicit` will be false. Otherwise, it will be true.

**dumper**
- `kind` (required) is a string identifier restricts type of acceptable objects.
  Allowed values are: "undefined", "null", "boolean", "integer", "float",
  "string", "array", "object", and "function".
- `instanceOf` (optional) allows to restrict acceptable objects with exactly one
  class. i.e. constructor function.
- `predicate` (optional) is a function of one argument. It takes an object and
  returns true to accept and false to discard.
- `representer` (optional) is a function intended to convert objects to simple,
  "dumpable" form. That is a string, an array, or a plain object.


### NIL

Special object used in type resolvers/representers to report failure of the
resolving/representing process. If your type handler cannot to process the given
object, it should return NIL.


## Supported YAML types

The list of standard YAML tags and corresponding JavaScipt types. See also
[YAML tag discussion](http://pyyaml.org/wiki/YAMLTagDiscussion) and
[YAML types repository](http://yaml.org/type/).

```
!!null ''                   # null
!!bool 'yes'                # bool
!!int '3...'                # number
!!float '3.14...'           # number
!!binary '...base64...'     # buffer
!!timestamp 'YYYY-...'      # date
!!omap [ ... ]              # array of key-value pairs
!!pairs [ ... ]             # array or array pairs
!!set { ... }               # array of objects with given keys and null values
!!str '...'                 # string
!!seq [ ... ]               # array
!!map { ... }               # object
```

**JavaScript-specific tags**

```
!!js/regexp /pattern/gim            # RegExp
!!js/undefined ''                   # Undefined
!!js/function 'function () {...}'   # Function
```


### Representing styles

```
!!null
  "canonical"   => "~"

!!int
  "binary"      => "0b1", "0b101010", "0b1110001111010"
  "octal"       => "01", "052", "016172"
  "decimal"     => "1", "42", "7290"
  "hexadecimal" => "0x1", "0x2A", "0x1C7A"

!!null, !!bool, !!float
  "lowercase"   => "null", "true", "false", ".nan", '.inf'
  "uppercase"   => "NULL", "TRUE", "FALSE", ".NAN", '.INF'
  "camelcase"   => "Null", "True", "False", ".NaN", '.Inf'
```

By default, !!int uses `decimal`, and !!null, !!bool, !!float use `lowercase`.


## Caveats

Note, that you use arrays or objects as key in JS-YAML. JS do not allows objects
or array as keys, and stringifies (by calling .toString method) them at the
moment of adding them.

``` yaml
---
? [ foo, bar ]
: - baz
? { foo: bar }
: - baz
  - baz
```
``` javascript
{ "foo,bar": ["baz"], "[object Object]": ["baz", "baz"] }
```

Also, reading of properties on implicit block mapping keys is not supported yet.
So, the following YAML document cannot be loaded.

``` yaml
&anchor foo:
  foo: bar
  *anchor: duplicate key
  baz: bat
  *anchor: duplicate key
```

## License

View the [LICENSE](https://github.com/nodeca/js-yaml/blob/master/LICENSE) file
(MIT).
