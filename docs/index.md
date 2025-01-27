# Dog's Roblox Luau Style Guide

This style guide aims to unify as much Lua code at Roblox as possible under the same style and conventions.

This guide is designed after the [Roblox Lua Style Guide](https://roblox.github.io/lua-style-guide/). I disagreed with some guidelines, so I made my own.

## Guiding Principles

-  The purpose of a style guide is to avoid arguments.
   -  There's no one right answer to how to format code, but consistency is important, so we agree to accept this one, somewhat arbitrary standard so we can spend more time writing code and less time arguing about formatting details in the review.
-  Optimize code for reading, not writing.
   -  You will write your code once. Many people will need to read it, from other developers, to any one else that touches the code, to you when you come back to it in six months.
-  Avoid magic, such as surprising or dangerous Luau features:
   -  Magical code is really nice to use, until something goes wrong. Then no one knows why it broke or how to fix it.
   -  Metatables are a good example of a powerful feature that should be used with care.
-  Be consistent with idiomatic Lua when appropriate.

## File Structure¶

    mkdocs.yml    # The configuration file.
    docs/
    	index.md  # The documentation homepage.
    	...       # Other markdown pages, images and other files.

I like to drink :beers: after I played :soccer:
