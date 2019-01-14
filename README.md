# Style-Guide-for-openResty-Code

We write code for the [LuaJIT](https://github.com/Kong/kong/issues/new)
interpreter, **not** Lua-PUC. As such, you should follow the LuaJIT best
practices:

- Do **not** instantiate global variables
- Consult the [LuaJIT wiki](http://wiki.luajit.org/Home)
- Follow the [Performance
  Guide](http://wiki.luajit.org/Numerical-Computing-Performance-Guide)
  recommendations
- Do **not** use [NYI functions](http://wiki.luajit.org/NYI) on hot code paths
- Prefer using the FFI over traditional bindings via the Lua C API
- Avoid table rehash by pre-allocating the slots of your tables when possible

  ```lua
  -- bad
  local t = {}
  for i = 1, 100 do
    t[i] = i
  end

  -- good
  local new_tab = require "table.new"
  
  local t = new_tab(100, 0)
  for i = 1, 100 do
    t[i] = i
  end
  ```

- Cache the globals used by your hot code paths

  ```lua
  -- bad
  for i = 1, 100 do
    t[i] = math.random()
  end

  -- good
  local random = math.random
  for i = 1, 100 do
    t[i] = random()
  end
  ```
  
- Cache the globals of `ngx.*`

  ```
  -- bad
  for i = 1, 100 do
    t[i] = ngx.var['arg_' .. i]
  end

  -- good
  local ngx_time = ngx.time
  for i = 1, 100 do
    t[i] = ngx_time()
  end
  ```

- Cache the length and indices of your tables to avoid unnecessary CPU cycles

  ```lua
  -- bad
  for i = 1, 100 do
    t[#t + 1] = other_tab[#other_tab]
  end

  -- good
  local n = 0
  local n_other_tab = #other_tab
  for i = 1, 100 do
    n = n + 1
    t[n] = other_tab[n_other_tab]
  end
  ```

And finally, most importantly: use your best judgment to design an
efficient algorithm. Doing so will always be more performant than a
poorly-designed algorithm, even following all the performance tricks of the
language you are using. :smile:

[Back to TOC](#table-of-contents)


## Code style

In order to ensure a healthy and consistent codebase, we ask of you that you
respect the adopted code style. This section contains a non-exhaustive list
of preferred styles for writing Lua. It is opinionated, but follows the
code styles of OpenResty and, by association, Nginx. OpenResty or Nginx
contributors should find themselves at ease when contributing to Kong.

- No line should be longer than 80 characters
- Indentation should consist of 2 spaces

When you are unsure about the style to adopt, please browse other parts of the
codebase to find a similar case, and stay consistent with it.

You might also notice places in the code base where the described style is not
respected. This is due to legacy code. **Contributions to update the code to
the recommended style are welcome!**

[Back to TOC](#table-of-contents)


### Table of Contents - Code style

- [Modules](#modules)
- [Variables](#variables)
- [Tables](#tables)
- [Strings](#strings)
- [Functions](#functions)
- [Conditional expressions](#conditional-expressions)


### Modules

When writing a module (a Lua file), separate logical blocks of code with
**two** blank lines:

```lua
local foo = require "kong.foo"


local _M = {}


function _M.bar()
  -- do thing...
end


function _M.baz()
  -- do thing...
end


return _M
```

[Back to code style TOC](#table-of-contents---code-style)


### Variables

When naming a variable or function, **do** use snake_case:

```lua
-- bad
local myString = "hello world"

-- good
local my_string = "hello world"
```

When assigning a constant variable, **do** give it an uppercase name:

```lua
-- bad
local max_len = 100

-- good
local MAX_LEN = 100
```

[Back to code style TOC](#table-of-contents---code-style)


### Tables

Use the constructor syntax, and **do** include a trailing comma:

```lua
-- bad
local t = {}
t.foo = "hello"
t.bar = "world"

-- good
local t = {
  foo = "hello",
  bar = "world", -- note the trailing comma
}
```

On single-line constructors, **do** include spaces around curly-braces and
assignments:

```lua
-- bad
local t = {foo="hello",bar="world"}

-- good
local t = { foo = "hello", bar = "world" }
```

[Back to code style TOC](#table-of-contents---code-style)


### Strings

When using the concatenation operator, **do** insert spaces around it:

```lua
-- bad
local str = "hello ".."world"

-- good
local str = "hello " .. "world"
```

[Back to code style TOC](#table-of-contents---code-style)


### Functions

Prefer the function syntax over variable syntax:

```lua
-- bad
local foo = function()

end

-- good
local function foo()

end
```

Perform validation early and return as early as possible:

```lua
-- bad
local function check_name(name)
  local valid = #name > 3
  valid = valid and #name < 30

  -- other validations

  return valid
end

-- good
local function check_name(name)
  if #name <= 3 or #name >= 30 then
    return false
  end

  -- other validations

  return true
end
```

Follow the return values conventions: Lua supports multiple return values, and
by convention, handles recoverable errors by returning `nil` plus a `string`
describing the error:

```lua
-- bad
local function check()
  local ok, err = do_thing()
  if not ok then
    return false, { message = err }
  end

  return true
end

-- good
local function check()
  local ok, err = do_thing()
  if not ok then
    return nil, "could not do thing: " .. err
  end

  return true
end
```

When a function call makes a line go over 80 characters, **do** align the
overflowing arguments to the first one:

```lua
-- bad
local str = string.format("SELECT * FROM users WHERE first_name = '%s'", first_name)

-- good
local str = string.format("SELECT * FROM users WHERE first_name = '%s'",
                          first_name)
```

[Back to code style TOC](#table-of-contents---code-style)


### Conditional expressions

Avoid writing 1-liner conditions, **do** indent the child branch:

```lua
-- bad
if err then return nil, err end

-- good
if err then
  return nil, err
end
```

When testing the assignment of a value, **do** use shortcuts, unless you
care about the difference between `nil` and `false`:

```lua
-- bad
if str ~= nil then

end

-- good
if str then

end
```

When creating multiple branches that span multiple lines, **do** include a
blank line above the `elseif` and `else` statements:

```lua
-- bad
if foo then
  do_stuff()
  keep_doing_stuff()
elseif bar then
  do_other_stuff()
  keep_doing_other_stuff()
else
  error()
end

-- good
if thing then
  do_stuff()
  keep_doing_stuff()

elseif bar then
  do_other_stuff()
  keep_doing_other_stuff()

else
  error()
end
```

For one-line blocks, blank lines are not necessary:

```lua
--- good
if foo then
  do_stuff()
else
  error("failed!")
end
```

Note in the correct "long" example that if some branches are long, then all
branches are created with the preceding blank line (including the one-liner
`else` case).

When a branch returns, **do not** create subsequent branches, but write the
rest of your logic on the parent branch:

```lua
-- bad
if not str then
  return nil, "bad value"
else
  do_thing(str)
end

-- good
if not str then
  return nil, "bad value"
end

do_thing(str)
```

When assigning a value or returning from a function, **do** use ternaries if
it makes the code more readable:

```lua
-- bad
local foo
if bar then
  foo = "hello"

else
  foo = "world"
end

-- good
local foo = bar and "hello" or "world"
```

When an expression makes a line longer than 80 characters, **do** align the
expression on the following lines:

```lua
-- bad
if thing_one < 1 and long_and_complicated_function(arg1, arg2) < 10 or thing_two > 10 then

end

-- good
if thing_one < 1 and long_and_complicated_function(arg1, arg2) < 10
   or thing_two > 10
then

end
```

[Back to code style TOC](#table-of-contents---code-style)
