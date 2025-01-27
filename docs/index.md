# Dog's Roblox Luau Style Guide

This style guide aims to unify as much Luau code as possible that I write, and others edit, under the same style and conventions.

This guide is designed after the [Roblox Lua Style Guide](https://roblox.github.io/lua-style-guide/). I disagreed with some guidelines, so I made my own.

Where the term file is mentioned, it can refer to any kind of `Script`.

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

1. An optional comment with the author's name and a brief description, if it's not obvious what the file does
1. Services used by the file, using `GetService`
    - Services should never be fetched anywhere else in the file, or by indexing `game`
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
- Consumers of libraries should require the API definition and then path to a public module, eg.
```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local MyLibrary = require(ReplicatedStorage.MyLibrary)
local MyModule = MyLibrary.MyModule
```

#### Example
For a project that looks like the following:
```
MyProject
|- FooBar
| |- Foo.lua
| |- Bar.lua
|- MyClass.lua
|- Packages
| |- Baz.lua
| | |- Bazifyer.lua
| | |- UnBazifyer.lua
```
MyClass should define the following import block:
```lua
-- 1. A definition of a common ancestor.
-- Use a relative path to make sure your project works in multiple locations!
local MyProject = script.Parent

-- 2. A block of all imported packages.
-- Baz is a library we depend on in our project, so we require its API directly...
local Baz = require(MyProject.Packages.Baz)

-- 3. A block for definitions derived from packages.
-- ...and then access its members through that API. These are simple so we don't need to break them down.
local Bazifyer = Baz.Bazifyer
local UnBazifyer = Baz.UnBazifyer

-- 4. A block for modules imported from the same project.
-- Defining the path to FooBar separately makes it faster to write and for others to read!
local FooBar = MyProject.FooBar
local Foo = require(FooBar.Foo)
local Bar = require(Foobar.Bar)
```

## Metatables
Metatables are an incredibly powerful Lua feature that can be used to overload operators, implement prototypical inheritance, and tinker with limited object lifecycle.
Ideally, metatables should be limited to a couple of cases:
- Implementing prototype-based classes
- Guarding against typos
- Using a table as a 'backup' in case a table doesn't have the specified key

### Prototype-based classes
There are multiple ways of defining classes in Lua. The method described below is recommended because it takes advantage of Luau's typing system. Providing a strongly-typed class definition helps developers use and improve your class by documenting its expected use, and allowing analysis tools and IDEs to warn against possible bugs when inconsistencies are detected.

First up, we create a regular, empty table:
```lua
local MyClass = {}
```
Next, we assign the __index member on the class back to itself. This is a handy trick that lets us use the class's table as the metatable for instances as well.

When we construct an instance, we'll tell Lua to use our __index value to find values that are missing in our instances. It's sort of like prototype in JavaScript, if you're familiar.
```lua
MyClass.__index = MyClass
```
In order to support strict type inference we are describing the shape of our class. This introduces some redundancy as we specify class members twice (once in the type definition, once as we build the actual instance), but warnings will be flagged if the two definitions fall out of sync with each other.
```lua
-- Export the type if you'd like to use it outside this module
export type ClassType = typeof(setmetatable(
    {} :: {
        property: number,
    },
    MyClass
))
```
Next, we create a default constructor for our class and assign the type definition from above to its return value (self).
```lua
-- The default constructor for our class is called `new` by convention.
function MyClass.new(property: number): ClassType
    local self = {
        -- Define members of the instance here, even if they're `nil` by default.
        property = property,
    }

    -- Tell Lua to fall back to looking in MyClass.__index for missing fields.
    setmetatable(self, MyClass)

    return self
end
```
We can also define methods that operate on instances. Prior to Luau's type analysis capabilities, popular convention has suggested using a colon (`:`) for methods. But in order to help the type checker understand that `self` has type `ClassType`, we use the dot (`.`) style of definition which allows us to specify the type of `self` explicitly. These methods can still be invoked on the resulting instances with a colon as expected.

In the future, [Luau will be able to understand the intended type of `self`](https://github.com/asajeffrey/rfcs/blob/shared-self-types/docs/shared-self-types.md) without any extra type annotations.
```lua
function MyClass.addOne(self: ClassType)
    self.property += 1
end
```
At this point, our class is ready to use!

We can construct instances and start tinkering with it:
```lua
local instance = MyClass.new(0)

-- Properties on the instance are visible, since it's just a table:
print(tostring(instance.property)) -- "0"

-- Methods are pulled from MyClass because of our metatable:
instance:addOne()
print(tostring(instance.property)) -- "1"
```
Further additions you can make to your class as needed:
- Introduce a `__tostring` metamethod to make debugging easier
- Define quasi-private members using two underscores as a prefix
- Add a method to check type given an instance, like:
```lua
function MyClass.isMyClass(instance)
    return getmetatable(instance).__index == MyClass
end
```

### Guarding against typos
Indexing into a table in Lua gives you `nil` if the key isn't present, which can cause errors that are difficult to trace!
Our other major use case for metatables is to prevent certain forms of this problem. For types that act like enums, we can carefully apply an `__index` metamethod that throws:
```lua
local MyEnum = {
    A = "A",
    B = "B",
    C = "C",
}

setmetatable(MyEnum, {
    __index = function(self, key)
        error(string.format("%q is not a valid member of MyEnum",
            tostring(key)), 2)
    end,
})
```
Since `__index` is only called when a key is missing in the table, `MyEnum.A` and `MyEnum.B` will still give you back the expected values, but `MyEnum.FROB` will throw, hopefully helping scripters track down bugs more easily.

## General Punctuation
- Don't use semicolons `;`. They are generally only useful to separate multiple statements on a single line, but you shouldn't be putting multiple statements on a single line anyway.

## General Whitespace
- **Indent with tabs.**
- Keep lines under 100 columns wide, assuming four column wide tabs.
- Wrap comments to 80 columns wide, assuming four column wide tabs.
    - This is different than normal code; the hope is that short lines help improve readability of comment prose, but is too restrictive for code.
- Don't leave whitespace at the end of lines.
    - If you're using an editor and it has an auto-trimming function, turn it on!
- Add a newline at the end of the file.
- No vertical alignment!
    - Vertical alignment makes code more difficult to edit and often gets messed up by subsequent editors.
    <p class="style-good">Good:</p>
    ```lua
    local frobulator = 132
    local grog = 17
    ```
    <p class="style-bad">Bad:</p>
    ```lua
    local frobulator = 132
    local grog       =  17
    ```

- Use a **single** empty line to express groups when useful. Do not start blocks with a blank line. Excess empty lines harm whole-file readability.
```lua
local Foo = require(Common.Foo)

local function gargle()
    -- gargle gargle
end

Foo.frobulate()
Foo.frobulate()

Foo.munt()
```

- Use one statement per line. Put function bodies on new lines.
    <p class="style-good">Good:</p>
    ```lua
    table.sort(stuff, function(a, b)
        local sum = a + b
        return math.abs(sum) > 2
    end)
    ```
    <p class="style-bad">Bad:</p>
    ```lua
    table.sort(stuff, function(a, b) local sum = a + b return math.abs(sum) > 2 end)
    ```
    This is especially true for functions that return multiple values. Compare these two statements:
    <p class="style-bad"></p>
    ```lua
    Rodux.Store.new(function(state) return state end, mockState, nil)
    Rodux.Store.new(function(state) return state, mockState end, nil)
    ```
    It's much easier to spot the mistake (and much harder to make in the first place) if the function isn't on one line.
    <p class="style-bad"></p>
    ```lua
    Rodux.Store.new(function(state)
        return state
    end, mockState, nil)
    Rodux.Store.new(function(state)
        return state, mockState
    end, nil)
    ```

    <p class="style-exception">Exception:</p>
    ```lua
    -- It's often faster and easier to read multiple guard clause if they are on one line.
    if valueIsInvalid then continue end
    ```

- Put a space before and after operators, except when clarifying precedence.
    <p class="style-good">Good:</p>
    ```lua
    print(5 + 5 * 6^2)
    ```
    <p class="style-bad">Bad:</p>
    ```lua
    print(5+5* 6 ^2)
    ```
- Put a space after each comma in tables and function calls.
    <p class="style-good">Good:</p>
    ```lua
    local friends = {"bob", "amy", "joe"}
    foo(5, 6, 7)
    ```
    <p class="style-bad">Bad:</p>
    ```lua
    local friends = {"bob","amy" ,"joe"}
    foo(5,6 ,7)
    ```