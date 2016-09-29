==========================
ES6 Symbols in Duktape 2.x
==========================

**WORK IN PROGRESS: Not sure if this approach works yet for ES6 semantics.**

Overview
========

Duktape 2.x adds ES6 Symbol support.  Duktape 1.x internal keys are unified
with the Symbol concept, and are considered a custom "hidden symbol" type
which is not visible to Ecmascript code.

The internal implementation is similar to existing internal keys.  Symbols
are represented as ``duk_hstring`` heap objects, with the string data
containing a byte prefix which is invalid (extended) UTF-8 so that it can
never occur for normal Ecmascript strings, or even strings with non-BMP
codepoints.  Object coerced strings have a special object class and the
underlying symbol is stored in ``_Value`` similarly to e.g. Number object.

See:

* https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Symbol

* http://www.2ality.com/2014/12/es6-symbols.html

Internal key formats
====================

Duktape custom hidden Symbols have an initial 0xFF byte prefix, which matches
the existing convention for Duktape 1.x internal keys.  While all bytes in the
range [0xC0,0xFE] are valid initial bytes for Duktape's extended UTF-8 flavor,
the continuation bytes [0x80,0xBF] are never a valid first byte so they are used
for symbols (and reserved for other future uses) in Duktape 2.x.

+-----------------------------------------------+-----------------------------------------------------------------+
| Internal string format                        | Description                                                     |
+-----------------------------------------------+-----------------------------------------------------------------+
| <ff> SomeUpperCaseValue                       | Existing Duktape internal properties, now called hidden symbols.|
|                                               | First byte is 0xFF, second is from [A-Z].                       |
+-----------------------------------------------+-----------------------------------------------------------------+
| <ff> anyOtherValue                            | Existing user internal properties, now called hidden symbols.   |
|                                               | First byte is 0xFF, second is ASCII (0x00-0x7f) but not         |
|                                               | from [A-Z].                                                     |
+-----------------------------------------------+-----------------------------------------------------------------+
| <ff> <ff> anyOtherValue                       | Existing user internal properties, now called hidden symbols.   |
|                                               | First and second bytes are 0xFF, remaining bytes arbitrary.     |
+-----------------------------------------------+-----------------------------------------------------------------+
| <a0> symbolDescription                        | Global symbol with description 'symbolDescription' created      |
|                                               | using Symbol.for().                                             |
+-----------------------------------------------+-----------------------------------------------------------------+
| <a1> symbolDescription <ff> uniqueSuffix      | Symbol with description 'symbolDescription' but with a trailing |
|                                               | unique string to make the symbol unique.  The unique suffix is  |
|                                               | opaque and chosen arbitrarily by Duktape.  It's unique within a |
|                                               | Duktape heap (across all global environments).                  |
+-----------------------------------------------+-----------------------------------------------------------------+
| <a1> <ff> uniqueSuffix                        | Anonymous symbol.  Unique suffix makes each such symbol unique. |
+-----------------------------------------------+-----------------------------------------------------------------+
| <a2 to bf>                                    | Initial bytes 0xA2 to 0xBF are reserved for future use.         |
+-----------------------------------------------+-----------------------------------------------------------------+

Useful comparisons (``p`` is pointer to string data) for internal use only:

* ``p[0] == 0xff || (p[0] & 0xc0) == 0x80``: some kind of Symbol, either Duktape
  hidden Symbol or an ES6 Symbol.

* ``p[0] == 0xff``: hidden symbol, user or Duktape

* ``(p[0] & 0xc0) == 0x80``: ES6 Symbol, visible to Ecmascript code

Global symbols
==============

Global symbols are the same across separate global environments and even across
Duktape heaps.  ES6 Section 19.4.2.1:

    The GlobalSymbolRegistry is a List that is globally available.
    It is shared by all Code Realms.

and ES6 Section 8.2:

    Before it is evaluated, all ECMAScript code must be associated with a Realm.
    Conceptually, a realm consists of a set of intrinsic objects, an ECMAScript
    global environment, all of the ECMAScript code that is loaded within the
    scope of that global environment, and other associated state and resources.

The current approach satisfies these simply by making a globally registered
Symbol have a fixed format so that if a Symbol with the same description is
created in another Duktape thread (or even Duktape heap), its internal
representation will be identical.  No explicit registry is maintained.

Unifying with Duktape internal keys
===================================

Necessary changes to add symbol behavior:

* Strings with initial byte 0xA0 or 0xA1 are also flagged as internal keys
  (``DUK_HSTRING_FLAG_INTERNAL``).  Lookup of the first byte indicates
  directly whether a symbol is hidden or not: 0xFF is hidden, any other is
  a normal ES6 Symbol.  Most likely useful to add a ``DUK_HSTRING_FLAG_ES6SYMBOL``
  flag.

* ``typeof(sym)`` should return "symbol" rather than string.  This should
  maybe happen for Duktape hidden symbols too.

* ``ToString(sym)`` needs special handling, similarly to how lightfuncs and
  plain buffers are handled.  String coercion code needs to strip possible
  "unique suffix" when coming up with the Symbol description.

* Symbols should be safe from accidental enumeration, JSON serialization, etc.
  This is actually already the case if symbols are treated as a special class
  of internal keys.

* Object.getOwnPropertySymbols() should return a list of symbol properties
  for an object, but filter out Duktape hidden symbols.

* ``Object(sym)`` should create an object with internal class "Symbol",
  with the plain symbol value stored behind ``_Value`` (hidden symbol
  property) as for Number objects, etc.

* Non-strict comparison needs to handle symbols.  ToPrimitive() coercion
  is maybe enough to ensure ``sym == Object(sym)`` is accepted.

* Property code needs to accept plain Symbols as is (treated like any other
  strings), and Symbol objects should look up their internal string value
  (instead of being coerced to e.g. ``Symbol(symbolDescription)``.  Current
  code just uses ``ToString()``.

* To check: is ``Symbol()`` exactly same as ``Symbol('')`` for semantics?

Some open questions
===================

How should C code see Symbols?
------------------------------

Easiest approach:

* Symbols are not enumerated by duk_enum() unless requested.  Either fold in with
  internal keys, or add a separate flags.  Maybe rename existing internal keys
  flag.

* Property operations work with symbols and internal keys without distinction.

* API call to create a symbol from C code.  Hides the construction of the internal
  string.

Best naming for Duktape internal keys
-------------------------------------

With https://github.com/svaarala/duktape/pull/979 Duktape internal properties
would become unreachable from Ecmascript code, even if you construct the
internal string using a buffer and then try to use it as an object key.
This offers more protection for sandboxing than ES6 Symbols which can be
enumerated.

What's the best name for there internal symbols?  They are a custom feature
but it's also possible Ecmascript will gain proper internal symbols at some
point.

Some possibilities:

* Internal symbol: easy to confuse with specification symbols for example.
  One benefit would be that as a term close to "internal property".

* Hidden symbol: conveys semantics (assuming GH-797) pretty well.

* Private symbol

* Native symbol

* Invisible symbol

Should Duktape 1.x internal keys just be considered ES6 Symbols?
----------------------------------------------------------------

Duktape 1.x doesn't provide true internal key hiding because it's possible to
lookup internal properties by constructing internal strings through buffers.

The easiest upgrade path would be to consider Duktape 1.x internal keys as
**ES6 symbols**, so that they'd be enumerable via ``Object.getOwnPropertySymbols()``.

Then add a specific prefix byte (say 0xA2) for actual hidden Symbols which can
be recommended for sandboxing.
