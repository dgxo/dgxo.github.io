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
    - Don't include spaces between the brackets and elements of a table.
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
- When creating blocks, inline any opening syntax elements.
    <p class="style-good">Good:</p>
    ```lua
    local foo = {
        bar = 2,
    }

    if foo then
        -- do something
    end
    ```

    <p class="style-bad">Bad:</p>
    ```lua
    local friends = {"bob","amy" ,"joe"}
    foo(5,6 ,7)
    ```
- Avoid putting curly braces for tables on their own line. Doing so harms readability, since it forces the reader to move to another line in an awkward spot in the statement.
    <p class="style-good">Good:</p>
    ```lua
    local foo = {
        bar = {
            baz = "baz",
        },
    }

    frob({
        x = 1,
    })
    ```
    <p class="style-bad">Bad:</p>
    ```lua
    local foo =
    {
        bar =

        {
            baz = "baz",
        },
    }

    frob(
    {
        x = 1,
    })
    ```
    <p class="style-exception">Exception:</p>
    ```lua
    -- In function calls with large inline tables or functions, sometimes it's
    -- more clear to put braces and functions on new lines:
    foo(
        {
            type = "foo",
        },
        function(something)
            print("Hello," something)
        end
    )

    -- As opposed to:
    foo({
        type = "foo",
    }, function(something) -- How do we indent this line?
        print("Hello,", something)
    end)
    ```

### Newlines in Long Expressions
- First, try and break up the expression so that no one part is long enough to need newlines. This isn't always the right answer, as keeping an expression together is sometimes more readable than trying to parse how several small expressions relate, but it's worth pausing to consider which case you're in.
- It is often worth breaking up tables and arrays with more than two or three keys, or with nested sub-tables, even if it doesn't exceed the line length limit. Shorter, simpler tables can stay on one line though.
- Prefer adding the extra trailing comma to the elements within a multiline table or array. This makes it easier to add new items or rearrange existing items.
- Break dictionary-like tables with more than a couple keys onto multiple lines.
    <p class="style-good">Good:</p>
    ```lua
    local foo = {type = "foo"}

    local bar = {
        type = "bar",
        phrase = "hooray",
    }

    -- It's also okay to use multiple lines for a single field
    local baz = {
        type = "baz",
    }
    ```
    <p class="style-bad">Bad:</p>
    ```lua
    local stuff = {hello = "world", hola = "mundo", howdy = "y'all", sup = "homies"}
    ```
- Break list-like tables onto multiple lines however it makes sense.
    - Make sure to follow the line length limit! 
    ```lua
    local libs = {"roact", "rodux", "testez", "cryo", "otter"}

    -- You can break these onto multiple lines, which makes diffs cleaner:
    local libs = {
        "roact",
        "rodux",
        "testez",
        "cryo",
        "otter",
    }

    -- We can also group them, if grouping has useful information:
    local libs = {
        "roact", "rodux", "cryo",

        "testez", "otter",
    }
    ```
- For long argument lists or longer, nested tables, prefer to expand all the subtables. This makes for the cleanest diffs as further changes are made.
    ```lua
        local aTable = {
        {
            aLongKey = aLongValue,
            anotherLongKey = anotherLongValue,
        },
        {
            aLongKey = anotherLongValue,
            anotherLongKey = aLongValue,
        },
    }

    doSomething(
        {
            aLongKey = aLongValue,
            anotherLongKey = anotherLongValue,
        },
        {
            aLongKey = anotherLongValue,
            anotherLongKey = aLongValue,
        }
    )
    ```

    In some situations where we only ever expect table literals, the following is acceptable, though there's a chance automated tooling could change this later. In particular, this comes up a lot in Roact code (`doSomething` being `Roact.createElement`).
    ```lua
    local aTable = {{
        aLongKey = aLongValue,
        anotherLongKey = anotherLongValue,
    }, {
        aLongKey = anotherLongValue,
        anotherLongKey = aLongValue,
    }}

    doSomething({
        aLongKey = aLongValue,
        anotherLongKey = anotherLongValue,
    }, {
        aLongKey = anotherLongValue,
        anotherLongKey = aLongValue,
    })
    ```
    However, this case is less acceptable if there are any non-tables added to the mix. In this case, you should use the style above.
    <p class="style-good">Good:</p>
    ```lua
    doSomething(
        {
            aLongKey = aLongValue,
            anotherLongKey = anotherLongValue
        },
        notATableLiteral,
        {
            aLongKey = anotherLongValue,
            anotherLongKey = aLongValue
        }
    )
    ```
    <p class="style-bad">Bad:</p>
    ```lua
    doSomething({
        aLongKey = aLongValue,
        anotherLongKey = anotherLongValue
    }, notATableLiteral, {
        aLongKey = anotherLongValue,
        anotherLongKey = aLongValue
    })
    ```
    - For long expressions try and add newlines between logical subunits. If you're adding up lots of terms, place each term on its own line. If you have parenthesized subexpressions, put each subexpression on a newline.
        - Place the operator at the beginning of the new line. This makes it clearer at a glance that this is a continuation of the previous line.
        - If you have to need to add newlines within a parenthesized subexpression, reconsider if you can't use temporary variables. If you still can't, add a new level of indentation for the parts of the statement inside the open parentheses much like you would with nested tables.
        - Don't put extra parentheses around the whole expression. This is necessary in Python, but Lua doesn't need anything special to indicate multiline expressions.
    - For long conditions in `if` statements, put the condition in its own indented section and place the `then` on its own line to separate the condition from the body of the `if` block. Break up the condition as any other long expression.
    <p class="style-good">Good:</p>
    ```lua
    if
        someReallyLongCondition
        and someOtherReallyLongCondition
        and somethingElse
    then
        doSomething()
        doSomethingElse()
    end
    ```
    <p class="style-bad">Bad:</p>
    ```lua
    if someReallyLongCondition and someOtherReallyLongCondition
        and somethingElse then
        doSomething()
        doSomethingElse()
    end

    if someReallyLongCondition and someOtherReallyLongCondition
            and somethingElse then
        doSomething()
        doSomethingElse()
    end

    if someReallyLongCondition and someOtherReallyLongCondition
        and somethingElse then
            doSomething()
            doSomethingElse()
    end
    ```

### if-then-else expressions
- Use `if-then-else` expressions over the `x and y or z` pattern for selecting a value. They're safer, faster and more readable.
    <p class="style-good">Good:</p>
    ```lua
    local scale = if someFlag() then 1 else 2
    ```
    <p class="style-bad">Bad:</p>
    ```lua
    local scale = someFlag() and 1 or 2
    ```
    - `if` expressions require an `else`. In some cases, we only use `someFlag()` and `someObject` without the `or`. It's fine to either leave this as is (it doesn't have the same safety issues) or convert it to `if someFlag() then someObject else nil`.
- Don't get carried away trying to fit everything into one statement though. These work best when they comfortably fit on one line.
- For multiple line `if` expressions, put the `then` and `else` at the start of new lines, each indented once.
    <p class="style-good">Good:</p>
    ```lua
    local scale = if someReallyLongFlagName() or someOtherReallyLongFlagName()
        then 1
        else 2
    ```
    <p class="style-bad">Bad:</p>
    ```lua
    local scale = if someReallyLongFlagName() or someOtherReallyLongFlagName() then 1
        else 2

    local scale = if someReallyLongFlagName() or someOtherReallyLongFlagName()
        then 1 else 2

    local scale = if someReallyLongFlagName() or someOtherReallyLongFlagName() then
        1 else 2
    ```
- If the `if` expression won't fit on three lines, convert it to a normal `if` statement.
    <p class="style-good">Good:</p>
    ```lua
    local scale
    if
        someReallyLongFlagName()
        or someOtherReallyLongFlagName()
    then
        scale = Vector2.new(1, 1) + someVectorOffset
            + someOtherVector
    else
        scale = Vector2.new(1, 1) + someNewVectorOffset
            + someNewOtherVector
    end
    ```
    <p class="style-bad">Bad:</p>
    ```lua
    local scale = if someReallyLongFlagName()
        or someOtherReallyLongFlagName()
        then Vector2.new(1, 1) + someVectorOffset
            + someOtherVector
        else Vector2.new(1, 1) + someNewVectorOffset
            + someNewOtherVector
    ```
- An exception to the above is if the `if` expression is in the middle of a much larger expression (e.g. a table definition or function call) and converting it to a normal `if` statement would involve copying a large number of lines.
    <p class="style-good">Good:</p>
    ```lua
    local thing = makeSomething("Foo", {
        OneChild = if someFlag()
            then makeSomething("Bar", {
                scale = 1,
            })
            else makeSomething("Bar", {
                scale = 2,
            }),
        TwoChild = makeSomething("Baz"),
    })
    ```
    <p class="style-bad">Bad:</p>
    ```lua
    local thing = makeSomething("Foo", {
        OneChild = if someFlag() then
            makeSomething("Bar", {
                scale = 1,
            })
        else
            makeSomething("Bar", {
                scale = 2,
            }),
        TwoChild = makeSomething("Baz"),
    })

    local thing = makeSomething("Foo", {
        OneChild = if someFlag() then makeSomething("Bar", {
            scale = 1,
        }) else makeSomething("Bar", {
            scale = 2,
        }),
        TwoChild = makeSomething("Baz"),
    })
    ```
- If the condition itself is too long to fit on one line, use a helper variable.
    <p class="style-good">Good:</p>
    ```lua
    local useNewScale = someReallyReallyLongFunctionName()
        and someOtherReallyLongFunctionName()
    local scale = if useNewScale then 1 else 2
    ```
    <p class="style-bad">Bad:</p>
    ```lua
    local scale = if someReallyReallyLongFunctionName()
        and someOtherReallyLongFunctionName()
        then 1
        else 2
    ```
- While `if` expressions do support `elseif`, it should be used sparingly. If your set of conditions is complicated enough to need several elseifs, then it may be difficult to read as a single expression. When using an `if` expression that includes `elseif` clauses is preferred, put the `elseif (condition)` then on a new line just like `then` and `else`.
    - This is a tradeoff. It would be more consistent to put the second then on a newline indented again, but then you end up deeply indented, which isn't good.
    ```lua
    local scale = if someFlag() then 1 elseif someOtherFlag() then 0.5 else 2

    local thing = makeSomething("Foo", {
        OneChild = if someFlag()
            then makeSomething("Bar", {
                scale = 1,
            })
            elseif someOtherFlag() then makeSomething("Bar", {
                scale = 0.5,
            })
            else makeSomething("Bar", {
                scale = 2,
            }),
        TwoChild = makeSomething("Baz"),
    })
    ```

## Blocks
- Don't use parentheses around the conditions in `if`, `while`, or `repeat` blocks. They aren't necessary in Lua!
    ```lua
    if CONDITION then
    end

    while CONDITION do
    end

    repeat
    until CONDITION
    ```
- Use `do` blocks if limiting the scope of a variable is useful.
    ```lua
    local getId
    do
        local lastId = 0
        getId = function()
            lastId = lastId + 1
            return lastId
        end
    end
    ```

## Literals
- Use double quotes when declaring string literals.
    - Using single quotes means we have to escape apostrophes, which are often useful in English words.
    - Empty strings are easier to identify with double quotes, because in some fonts two single quotes might look like a single double quote (`""` vs `''`).
    <p class="style-good">Good:</p>
    ```lua
    print("Here's a message!")
    ```
    <p class="style-bad">Bad:</p>
    ```lua
    print('Here\'s a message!')
    ```
    - Single quotes are acceptable if the string contains double quotes to reduce escape sequences.
    <p class="style-exception">Exception:</p>
    ```lua
    print('Quoth the raven, "Nevermore"')
    ```
    - If the string contains both single and double quotes, prefer double quotes on the outside, but use your best judgement.

## Tables
- Avoid tables with both list-like and dictionary-like keys.
    - Iterating over these mixed tables is troublesome.
- Don't specify `pairs` or `ipairs` as the iterator when iterating over a table. Luau supports `for key, value in table` syntax, which is generally more readable.
    - The argument that this helps clarify what kind of table we're expecting is irrelevant with types annotations.
- Add trailing commas in multi-line tables.
    - This lets us re-sort lines with a single keypress (++Alt+up++ and ++Alt+down++).
    ```lua
    local frobs = {
        andrew = true,
        billy = true,
        caroline = true,
    }
    ```

## Functions
Keep the number of arguments to a given function small, preferably 1 or 2.

Always use parentheses when calling a function. Lua allows you to skip them in many cases, but the results are typically much harder to parse.