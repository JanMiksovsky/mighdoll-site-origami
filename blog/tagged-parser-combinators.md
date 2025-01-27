### Tagged Parser Combinators

I added a little **tag** feature to the
[mini-parse] parser combinator library.
Tagging elements in a grammar helps
users reliably extract particular elements from a parse tree,
even a deeply nested parse tree.
I haven't seen tags in other parser combinator libraries,
but it's probably not
too hard to add to your favorite parsing library if iT
doesn't already use tags.

If you've used a parser combinator library before,
you'll probably have run into the problem
of extracting the values you want from the parse tree.
Typically, complex parser results are returned as arrays
and nested arrays of values.
Selecting the value you want from the results by indexing
into these arrays is feasible but fragile.
As we'll see, tagging results makes extraction more convenient and maintainable.

Perhaps the most interesting part of
building the **tag** feature involved TypeScript
type system tricks. See the
[Typing Tagged Results](#typing-tagged-results) section
below for links to TypeScript details.

### Parser combinators

Parser combinators allow defining a grammar using
a library of functions and methods in TypeScript.
Each parser is a TypeScript object
that represents a part of the grammar
and offers a `parse` method to read input text
and produce structured output.
The library makes it easy to combine parsers into more complex parsers.
Independent parsers are easy to test and reuse.
And implementation as a runtime TypeScript library
makes for lightweight integration of parsing into applications.

Here's a small example.
Say we want to parse a programming language with
function declarations like this: `fn foo()`.
With a parser combinator library like [mini-parse],
you can define the grammar to parse simple function declarations as follows:

```ts
import { seq, kind } from "mini-parse";
const ident = kind("ident");

// ---cut---
const fnDecl = seq("fn", ident, "(", ")");
```

Here `seq` is a parser combinator function provided by the library.
`seq` returns a new parser that sequentially runs a series of provided parsers.
`seq` handles string arguments by converting them into parsers that match
the provided string.
So `fnDecl` is a parser that matches a string "fn" followed by
an identifier, followed by a "(", followed by a ")".

Here's a complete example. Runnable version [TODO].

```ts
import { seq, kind, matchOneOf, matchingLexer, tokenMatcher } from "mini-parse";

// ---cut---
const src = "fn foo()";

// lexer
const tokens = tokenMatcher({
  ident: /[a-z]+/,
  ws: /\s+/,
  symbol: matchOneOf("( ) [ ] { } ; ,"),
});
const lexer = matchingLexer(src, tokens);

// parsers
const ident = kind(tokens.ident);
const fnDecl = seq("fn", ident, "(", ")");

// run parser
const result = fnDecl.parse({ lexer });

// extract identifier from parsed result
if (result) {
  const foundIdent = result.value[1]; // magic number #1
  console.log(`found fn name: ${foundIdent}`);
}
```

### Dangerous extraction

Extracting the results by indexing to position
1 in the results isn't so great from a maintenance point of view.
If the grammar changes, the index will change, and our code will break.

Let's say we update the grammar to allow for optional annotations,
to allow the possibility of something like `@export fn foo()`.

```ts
import { seq, opt, kind, matchingLexer, tokenMatcher } from "mini-parse";

const ident = kind("ident");

// ---cut---
const annotation = opt(seq("@", ident));
const fnDecl = seq(annotation, "fn", ident, "(", ")");
```

Now the result we want is at magic number #2
and the extraction code above won't work anymore.

In a more complicated case,
we might be looking for elements deeper in the parse tree,
requiring us to index multiple times and leaving
us vulnerable to restructuring of the parsers even if we don't
add any elements to the source language.
Or we might want to extract multiple similar values,
possibly from multiple places in the parsed results,
further complicating our value extraction.

Indexing into the results works, but it's fragile.
And the maintenance risk grows if the language we're parsing
evolves over time..

### Tagging results

Instead of indexing into results,
let's add a `tag` feature to the combinator library
to identify the results we care about.
Then we can collect the results by name, rather than by index.

It'll look like this.

```ts
import { seq, kind, opt, repeat } from "mini-parse";
const ident = kind("ident");

// ---cut---
const annotation = repeat(seq("@", ident).tag("annotated")); // tag annotations
const fnDecl = seq(annotation, "fn", ident.tag("fnName"), "(", ")"); // tag fnName
```

While we're arranging for tagging, we'll have the tagged values accumulate
into an array so we can collect multiple matches.
e.g. maybe there are multiple annotations in this case.
And we'll have the tagged values propagate up the
and parse tree for easy collection.

In this case, the potentially multiple annotations are collected into an array in
the `annotation` parser under the tag `annotated`.
The `annotated` tag results also propagate to the parent `fnDecl` parser too.

```ts
import { seq, kind, opt, repeat } from "mini-parse";
const ident = kind("ident");

const annotation = repeat(seq("@", ident.tag("annotated"))); // tag annotations
const fnDecl = seq(annotation, "fn", ident.tag("fnName"), "(", ")"); // tag fnName

fnDecl.map((result) => {
  // ---cut---
  const [fnName] = result.tags.fnName;
  const annotations: string[] = result.tags.annotated;
  // ---cut-after---
});
```

Saving tags in a combinator library isn't much work, and
tagging helps make extracting values from the parser more convenient and
maintainable for users of the library.
No magic numbers, no maintenance problems when updating the grammar.

Consider adding tags to your favorite Parser Combinator library too.

### Typing Tagged Results

We could have a general type of `Record<string, any[]>` for the result tags.
That works ok, but we can do better.

It'll be more useful to the user of the library if TypeScript
understands that our tags for particular parsers.
e.g. for `fnParser` the tags should be typed: `{ fnName: string[]; annotated: string[] }`.
If TypeScript understands the tags, autocomplete in the editor will be smart,
and the compiler will catch typos in the tag names,
mistakes in object types, etc.

Good types for tags requires some tricky work with
the TypeScript type system, though.
Let's set up the problem here.

The basic idea is that every parser will have two type parameters,
one for a result of its parse, and one for the accumulated tags.
Something like this:

```ts
export type TagRecord = Record<string | symbol, any[]>;
class Parser<V, T extends TagRecord> {}
```

Parser gets a `tag` method that takes a string and returns a new parser:

```ts
// @noErrors
class Parser<V, T extends TagRecord> {
  tag( tagName: string ): Parser<V, ???>
//                                  ^^^
}
```

And combinators that compose parsers will need compose the TagRecord types too.

```ts
// @noErrors
function seq(...parsers: Parser<any, any>[] ): Parser<???, ???> {}
//                                                    ^^^  ^^^
```

The challenge is to fill in those `???` types in TypeScript.
For a detailed explanation
of the TypeScript tricks read [TypeScript Type Tricks For Records].
Or go straight for [CombinatorTypes.ts] for the implementation in [mini-parse].

[Mini-Parse]: https://npmjs.com/package/mini-parse
[TypeScript Type Tricks For Records]: https://github.com/mighdoll/wgsl-linker/tree/main/packages/mini-parse/doc/TypeTricksForRecords.md
[CombinatorTypes.ts]: https://github.com/mighdoll/wgsl-linker/tree/main/packages/mini-parse/src/CombinatorTypes.ts
