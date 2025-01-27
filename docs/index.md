# Dog's Roblox Luau Style Guide

This style guide aims to unify as much Lua code at Roblox as possible under the same style and conventions.

This guide is designed after the [Roblox Lua Style Guide](https://roblox.github.io/lua-style-guide/). I disagreed with some guidelines, so I made my own.

Where files are mentioned, these can refer to any kind of `Script`.

## Guiding Principles

- The purpose of a style guide is to avoid arguments.
    - There's no one right answer to how to format code, but consistency is important, so we agree to accept this one, somewhat arbitrary standard so we can spend more time writing code and less time arguing about formatting details in the review.
- Optimize code for reading, not writing.
    - You will write your code once. Many people will need to read it, from other developers, to any one else that touches the code, to you when you come back to it in six months.
- Avoid magic, such as surprising or dangerous Luau features:
    - Magical code is really nice to use, until something goes wrong. Then no one knows why it broke or how to fix it.
    - Metatables are a good example of a powerful feature that should be used with care.
- Be consistent with idiomatic Lua when appropriate.

## File Structure
Files should consist of these things (if present) in order:

1. An optional comment with the author's name and a brief description, if it's not obvious what the script does
1. Services used by the script, using `GetService`
1. Module imports, using `require`
1. Constants
1. Variables and functions
1. (if module) The object the module returns
1. (if module) A return statement

## Requires
### General
- All require calls must be at the top of a file, making dependencies static.
- Files with a lot of requires should have them be sorted alphabetically, by module name.

### Requiring Libraries
Libraries are projects which define an API for external consumers to use, typically by providing a top-level table which requires other modules. Libraries will typically provide a structured public API composed from internal modules. This allows libraries to have stable interfaces even when internal details may change, and can be used both for sharing code as well as for organizing one's own code.

- Library internals should require their public and private modules directly, eg.
```lua
-- in MyLibrary/Foo.lua
local MyLibrary = script.Parent
local MyModule = require(MyLibrary.MyModule)
```