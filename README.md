# Literate Haskell support for Markdown

`markdown-unlit` is a custom `unlit` program.  It can be used to extract
Haskell code from Markdown files.  It also supports [tangling](#tangling),
which lets you define named code fragments and assemble them by reference.

To use it with GHC, add

    ghc-options: -pgmL markdown-unlit

to your cabal file.

## Extended example

> tl;dr `markdown-unlit` allows you to have a `README.md`, that at the
> same time is a literate Haskell program.

The following steps show you how to set things up, so that:

 * the Haskell code in your `README.md` gets syntax highlighted on GitHub
 * you can run your literate Haskell within GHCi
 * you can create a Cabal `test-suite` from your `README.md` (no broken code
   examples anymore, yay!)

The complete code of this example is provided in the [`example`](https://github.com/sol/markdown-unlit/tree/main/example) subdirectory.

### 1. Install `markdown-unlit`

    $ cabal update && cabal install markdown-unlit


### 2. Create a `README.md`


    # nifty-library: Do nifty things (effortlessly!)

    Here is a basic example:

    ```haskell
    main :: IO ()
    main = putStrLn "That was easy!"
    ```

Code blocks with class `haskell` are syntax highlighted on GitHub ([like
so](https://github.com/sol/markdown-unlit/blob/main/example/README.md#readme)).

### 3. Create a symbolic link `README.lhs -> README.md`

    $ ln -s README.md README.lhs

### 4. Run your code

At this point we can load the code into GHCi:

    $ ghci -pgmL markdown-unlit README.lhs
    *Main> main
    That was easy!

Or better yet, pipe the required flag into a `.ghci` file, and forget about it:

```
$ echo ':set -pgmL markdown-unlit' >> .ghci
```
```
$ ghci README.lhs
*Main> main
That was easy!
```

### 5. Create a Cabal `test-suite`

```
name:             nifty-library
version:          0.0.0
build-type:       Simple
cabal-version:    >= 1.8

library
  -- nothing here yet

test-suite readme
  type:           exitcode-stdio-1.0
  main-is:        README.lhs
  build-depends:  base, markdown-unlit
  ghc-options:    -pgmL markdown-unlit
  build-tool-depends: markdown-unlit:markdown-unlit
```

Run it like so:

    $ cabal configure --enable-tests && cabal build && cabal test

## Reordering code blocks

Code blocks that are tagged with `top` are moved to the beginning of the source
file.

**Example:**

    ## Introduction

    ```haskell top
    {-# LANGUAGE OverloadedStrings #-}
    module MyModule where
    ```

    ## Working with textual data

    ```haskell top
    import qualified Data.Text as Text
    ```
    ```haskell
    -- |
    -- >>> foo
    -- 3
    foo :: Int
    foo = Text.length "foo"
    ```

    ## Working with binary data

    ```haskell top
    import qualified Data.ByteString as ByteString
    ```
    ```haskell
    -- |
    -- >>> bar
    -- 3
    bar :: Int
    bar = ByteString.length "foo"
    ```

~~~
$ doctest -pgmL markdown-unlit MyModule.lhs
Examples: 2  Tried: 2  Errors: 0  Failures: 0
~~~


### More fine-grained control over the ordering

Optionally, `top` can be followed by `:n` where `n` is a non-negative integer.
Code blocks with a smaller `n` move above code blocks with a larger `n`.
`top` is an alias for `top:0`.

## Customizing

By default, `markdown-unlit` extracts all code that is marked with `haskell`,
unless it is marked with `ignore` as well.  You can customize this by passing
`-optL <pattern>` to GHC.

A simple pattern is just a class name, e.g.:

    -optL foo

extracts all code that is marked with `foo`.

A class name can be negated by prepending it with a `!`, e.g.

    -optL !foo

extracts all code, unless it is marked with `foo`.

You can use `+` to combine two patterns with *AND*, e.g.

    -optL foo+bar

extracts all code that is marked with both `foo` and `bar`.

If `-optL` is given multiple times, the patterns are combined with *OR*, e.g.

    -optL foo -optL bar

extracts all code that is either marked with `foo` or `bar`.

## Tangling

Within extracted code, lines of the form `-- #name` act as named block
definitions or references, letting you present code in any order and assemble
it during extraction.

A **definition** is a `-- #name` line that appears after a blank line (or at the
start of a code block) and is followed by one or more body lines:

    ```haskell
    -- #greet
    putStrLn "Hello!"
    ```

A **reference** is a `-- #name` line that appears anywhere else.  It is replaced
by the named block's body, with the reference line's leading whitespace
prepended to each expanded line:

    ```haskell
    main :: IO ()
    main = do
        -- #greet
        -- #farewell
    ```

Multiple definitions with the same `#name` (even across different code blocks)
are concatenated in file order.  References may be nested: a definition body
can itself contain references to other named blocks.  Circular references are
detected and reported as errors.

GHC `#line` directives in the output point to each line's original position in
the Markdown source, so compiler errors remain accurate after expansion.

**Example:**

Given this Markdown file:

    ## Main

    ```haskell
    module Main where

    -- #imports

    main :: IO ()
    main = do
        -- #greet
        -- #farewell
    ```

    ## Greetings

    ```haskell
    -- #imports
    import Data.List (intercalate)
    ```

    ```haskell
    -- #greet
    putStrLn (intercalate ", " ["Hello", "world!"])
    ```

    ## Farewells

    ```haskell
    -- #farewell
    putStrLn "Goodbye!"
    ```

The extracted output is equivalent to:

```haskell
module Main where

import Data.List (intercalate)

main :: IO ()
main = do
    putStrLn (intercalate ", " ["Hello", "world!"])
    putStrLn "Goodbye!"
```

## Development

### Limitations

 * [indented code blocks](http://daringfireball.net/projects/markdown/syntax#precode) are not yet supported

If you want to get any limitation lifted, open a ticket or send a pull request.

### Contributing

Add tests for new code, and make sure that the test suite passes with your
modifications.

    cabal configure --enable-tests && cabal build && cabal test

## Real world examples

 * [attoparsec-parsec](https://github.com/sol/attoparsec-parsec#readme)
 * [hspec-expectations](https://github.com/sol/hspec-expectations#readme)
 * [wai](https://github.com/yesodweb/wai/tree/master/wai#readme)
 * [servant-tutorial](https://github.com/haskell-servant/servant/tree/master/doc/tutorial)
 * [servant-cookbook](https://github.com/haskell-servant/servant/tree/master/doc/cookbook)
 * [ClickHaskell-usage](https://github.com/KovalevDima/ClickHaskell/tree/master/usage)
 * [ClickHaskell-contribution](https://github.com/KovalevDima/ClickHaskell/tree/master/contribution)
 * [ClickHaskell-testing](https://github.com/KovalevDima/ClickHaskell/tree/master/testing)

That's it, have fun!
