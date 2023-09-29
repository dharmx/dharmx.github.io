---
draft: true

title: "Neovim Plugin Manager"
subtitle: "A minimal package manager for Neovim. Sample configuration included."
summary: "Decided to write a package manager for Neovim within 100 lines of code.
          Turned out pretty well in my opinion. Additionally, features that could
          not be included are also documented and included."
description: "Minimal package manager for Neovim."

date: 2022-08-24T17:29:27+05:30
lastmod: 2022-08-24T17:29:27+05:30

author: "dharmx"
authorLink: "https://dharmx.is-a.dev"
license: "<a rel='license external nofollow noopener noreffer' href='https://opensource.org/licenses/GPL-3.0' target='_blank'>GPL-3.0</a>"

tags: ["nvim", "lua"]
categories: ["deepdive"]
relative: true

featuredImage: "images/featured-image.png"
featuredImagePreview: "images/featured-image.png"

images: ["/nvim-plugin-manager/images/featured-image.png"]
toc:
  auto: true

hiddenFromHomePage: false
hiddenFromSearch: false
twemoji: false
lightgallery: false
ruby: false
fraction: false
fontawesome: false
linkToMarkdown: true
rssFullText: false
---

# Introduction

Hello again. It's been quite a while since my wrote my last post. Been really busy with studies and whatnot.
This time I'll be writing some code for once. We'll be creating a nice and simple plugin manager for Neovim.
This will help in getting a deeper and clearer understaning of how plugin managers actually work in Neovim.

{{< admonition warning "Code Quality." false >}}
Please, keep in mind that the code may end up looking really cluttered. This is because I am optimizing towards
lines of code. So, I'll probably end up squeezing as much stuff as humanely possible.
{{< /admonition >}}

## Plan

Currently, I am planning to add the following features.

- Define a nice API for adding plugin specifications.
- Allow downloading to `opt` to support Vim's default lazy loading mechanism.
- Allow `PluginLoadPost`, `PluginInstallPost` and `PluginUpdatePost` auto-commands.
- Store `installed`, `location`, `link`, `loaded`, `name`, and `username` metadata keys.

Additionally, I will be implementing more advanced lazy handlers but won't be adding them to the plugin code.
Instead, I will add an "Extras" section and add code snippets with minimal descriptions there. Lastly, I will
also attach a minimal nvim configuration file for a demonstration.

{{< admonition info "Before we proceed." >}}
This is a fairly advanced Neovim/Lua article. Therefore, moderate familiarity with the Lua programming language
is required. Additionally, some familiarity with Neovim Lua API is preferred. I will also simplify what each
Neovim API does along the way as best as I can.
{{< /admonition >}}

## Metatables

I will mainly be explaining how the add, sub, index, call and newindex meta-methods work.
Out of this the "add" meta-method will be the backbone of this plugin manager so pay extra attention!

Now, for starters, just know that metatables allow us to change the behaviour of a table.
It does so by using specific events i.e. say you want to add two items `a + b` where `a` and `b` are tables.
Whenever Lua tries to add two tables, it checks whether either of them has a metatable and whether that
metatable has an `__add` field. If Lua finds this field, it calls the corresponding value
(the so-called meta-method, which should be a function) to compute the sum.
If you have enough experience with metatables then you may skip to the
[spec]({{< ref "#plugin-specification" >}}) section.

### <samp>__add</samp>

It defines the binary add operation for a table. For example, let us take a table called `manga` which
will contain five websites where you can read [manga](https://en.wikipedia.org/wiki/Manga).

```lua
local MANGA = {
  "mangadex.org",
  "manganato.com",
  "cubari.moe",
  "webtoons.com",
  "reaperscans.com",
}
```

Say you want to add more of said sites. In that case you need to use `table.insert(MANGA, "fanfox.net")`.
Moving further now you need to add ten more of such sites. What will you do then?
Of course you can create a wrapper function around `table.insert` to make this a bit cleaner.

```lua
local add_manga = function(manga) table.insert(MANGA, manga) end
add_manga("fanfox.net")
add_manga("wuxiaworld.com")
...
```

This is acceptable but this could be a lot cleaner if you use a metatable. And, by doing that
you can simply add sites to the table by `MANGA = MANGA + "site.com" + "example.com"`. For this
we will be using the `setmetatable` builtin to override the default `+` operation.

```lua
local _META = {}

---@param this table the MANGA table.
---@param new string the new entry to be added to MANGA.
_META.__add = function(this, new)
  assert(new, "new cannot be nil")
  table.insert(this, new)
  return this
end
setmetatable(MANGA, _META)

MANGA = MANGA + "mangakakalot.to" + "mangareader.to" + "readm.org"
vim.print(MANGA) -- run by :luafile %
```

This will result on the following output.

```sh
{
  "mangadex.org",
  "manganato.com",
  "cubari.moe",
  "webtoons.com",
  "reaperscans.com",
  "mangakakalot.to",
  "mangareader.to",
  "readm.org",
  <metatable> = {
    __add = <function 1>
  }
}
```

### <samp>__sub</samp>

This is the opposite of the `__add` metatable. This overrides the `-` operator.
Now, consider the following example (we will be using the same example as `__add`).

```lua
_META.__sub = function(this, other)
  assert(other, "other should not be nil")
  for index, item in ipairs(this) do
    if item == other then
      table.remove(this, index)
      return this
    end
  end
  return this
end
setmetatable(MANGA, _META)

MANGA = MANGA + "mangakakalot.to" + "mangareader.to" + "readm.org"
MANGA = MANGA - "webtoons.com" + "tapas.io" - "reaperscans.com"
vim.print(MANGA)
```

This will result in the following.

```sh
{
  "mangadex.org",
  "manganato.com",
  "cubari.moe",
  "mangakakalot.to",
  "mangareader.to",
  "readm.org",
  "tapas.io",
  <metatable> = {
    __add = <function 1>,
    __sub = <function 2>
  }
}
```

### <samp>__index</samp>

This one's a lot easier than the ones we have seen so far. This meta-method runs when you try to index a `nil`
element of the table this metatable is connected to, `table[index]`. Hence, consider the following example.

```lua
-- _META.__index = function(...) ... end will only work if an element does not exist in the MANGA table.
-- Therefore, we will do the following instead. We will be creating a proxy table.
local _EXTRA = setmetatable({}, {
  __index = function(_, key)
    local item = MANGA[key] -- gets from outer scope
    if item then
      return {
        name = item,
        length = item:len(),
        domain = vim.split(item, ".", { plain = true })[2]
      }
    end
  end
})
vim.print(_EXTRA[2])
```

This one's output will be as follows.

```sh
{
  domain = "com",
  length = 13,
  name = "manganato.com"
}
```

### <samp>__newindex</samp>

This is similar to the `__index` metatable. This meta-method runs when you try to index and set a `nil` element of the
table this metatable is connected to, `table[index] = value`. Now, see the following example.

```lua
_META.__newindex = function(this, key, value)
  local message = "%s.%s=%s entry is not allowed. New entries are not allowed."
  vim.api.nvim_err_writeln(string.format(message, this, key, value))
end

setmetatable(MANGA, _META)
MANGA._name = "manga"
```

You will see the following output.

```txt
[table: 0x7f9f536cb150]._name=manga entry is not allowed. New entries are not allowed.
```

### <samp>__call</samp>

This is probably the most confusing metatable of them all. This metatable basically allows you to
call a table. Yes. It turns a table into a table-function. If you try to call the `MANGA` table will
result in a `MANGA.__call(...)` call. Now, take a look at a more legible example.

```lua
_META.__call = function(this, items)
  local type_items = type(items)
  assert(type_items == "table" or type_items == "string", "items can be string/table")
  if type(items) == "string" then items = { items } end
  for _, item in ipairs(items) do this = this - item + item end
  return this
end
setmetatable(MANGA, _META)
vim.print(MANGA("hello.com"))
vim.print(MANGA({ "garbage.com", "wuxiaworld.com", "gogomanga.com" }))
```

This will yield the following output.

```sh
{
  "mangadex.org",
  "manganato.com",
  "cubari.moe",
  "mangakakalot.to",
  "mangareader.to",
  "readm.org",
  "tapas.io",
  "hello.com",
  <metatable> = {
    __add = <function 1>,
    __call = <function 2>,
    __newindex = <function 3>,
    __sub = <function 4>
  }
}
{
  "mangadex.org",
  "manganato.com",
  "cubari.moe",
  "mangakakalot.to",
  "mangareader.to",
  "readm.org",
  "tapas.io",
  "hello.com",
  "garbage.com",
  "wuxiaworld.com",
  "gogomanga.com",
  <metatable> = {
    __add = <function 1>,
    __call = <function 2>,
    __newindex = <function 3>,
    __sub = <function 4>
  }
}
```

## Domain Info

Now, that the Lua stuff is out of the way, it is time for a primer on some Neovim/Vim API and elements that
we will be using.

### <samp>vim.o</samp> and <samp>vim.opt</samp>

These are the options that you set with the `:set` command. The difference between `vim.o` and `vim.opt` is
that one is easy to use (as in getting its value and setting its value). The reason being that all elements of `vim.o`
are string indexed and values are not an array. It is either `string` or `number`. These do not use `boolean`,
`table`, `nil`, etc. But, on the other hand `vim.opt` is easy to work with if you need to safely `append`, `prepend`,
or `remove` from a list-like setting. So, take `vim.o.fillchars` for example - its value (for me) looks like the
following.

```lua
"diff:░,eob: ,fold:─,foldclose:,foldopen:,foldsep:│,horiz:━,horizdown:┳,horizup:┻,msgsep:━,stlnc: "`
```

whereas, if you inspect `vim.opt.fillchars` instead, you'll get something like the following.

```lua
{
  _info = {
    allows_duplicates = false,
    commalist = true,
    default = "",
    flaglist = false,
    global_local = true,
    last_set_chan = 0,
    last_set_linenr = 0,
    last_set_sid = 0,
    metatype = "map",
    name = "fillchars",
    scope = "win",
    shortname = "fcs",
    type = "string",
    was_set = false
  },
  _name = "fillchars",
  _value = "diff:░,eob: ,fold:─,foldclose:,foldopen:,foldsep:│,horiz:━,horizdown:┳",
  <metatable> = <1>{
    __add = <function 1>,
    __index = <table 1>,
    __pow = <function 2>,
    __sub = <function 3>,
    _set = <function 4>,
    append = <function 5>,
    get = <function 6>,
    prepend = <function 7>,
    remove = <function 8>
  }
}
```

See the meta-methods. This implements `__add` and `__sub`. Additionally, there are multiple info (`_info`)
about the current state of the setting as well (which you might need later down the line).
Now, look at the following code snippet.

```lua
-- this is not really pleasant to look at
vim.o.fillchars = vim.o.fillchars .. ",verthoriz:╋,vertleft:┫,vertright:┣"
-- this is a lot cleaner and legible
vim.opt.fillchars = vim.opt.fillchars + "verthoriz:╋" + "vertleft:┫" + "vertright:┣"
vim.opt.rtp:prepend("/home/username/.bin/local_plugin") -- rtp -> runtimepath

-- alternative vim.opt.rtp = vim.opt.rtp + "/home/username/.bin/local_plugin"
vim.opt.runtimepath:append("/home/username/.bin/local_plugin")
vim.print(vim.opt.fillchars) -- NOTE: This will print a table instead.

-- these are a bit scuffed
vim.print(vim.opt.fillchars._value) -- same
vim.print(vim.opt.fillchars:get()) -- same

-- INFO: If all you want is to inspect the current value of a setting then you may prefer vim.o instead.
vim.print(vim.o.fillchars) -- same as vim.opt.fillchars:get()
```

I hope the difference is clear now.

### packages and packpath

Please read the <samp>[runtimepath](https://vimhelp.org/options.txt.html#%27runtimepath%27)</samp> help page section.
Then the <samp>[packpath](https://vimhelp.org/options.txt.html#%27packpath%27)</samp> section and finally, the
<samp>[packages](https://vimhelp.org/repeat.txt.html#packages)</samp> section. I am too lazy to explain this. I'll
edit and write my own explanations and examples when I'm in mood.
Finally, let us quickly move to the main part 🤩!

## Plugin Specification

For starters, please skim through this [nvim-package-specification](https://github.com/nvim-lua/nvim-package-specification) document.
This is an unofficial draft but it is useful for getting a gist of what this section will be about.
We will only _adapt_ the `package` and `source` from this. And, some of custom keys will also be added!
Hence, please take a look at the brief description of the new hybrid specification below.

- `[1]`: The source of the plugin. The main difference from the draft is that this will use [URI](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier)
  And, it will effectively combine the `name` and the `url` keys from the draft.
- `installed`: A key that will be set by the plugin itself after the plugin is downloaded to the `location` path.
  Additionally, the plugin manager will also check if the plugin is already installed and set that field accordingly.
- `loaded`: This will be similar to `loaded`. The difference: It will be set to `true` after
  <samp>[:packadd](https://neovim.io/doc/user/repeat.html#%3Apackadd)</samp>.

## Events

Most plugin managers have plugin specification keys like `config`, `build`, `dependencies`, etc.
But, we won't be implementing them strictly here. Instead, we will try to leverage auto-commands
as much as possible.

The core concept is for example, fire an user event when a plugin is loaded. We can then bind a
callback to that event which will check which plugin has been loaded based on the metadata passed
into the callback and then load its configuration. For making this concerte we will implement the
following events.

- `PluginLoadPost`: Fired when a plugin has been loaded.
- `PluginInstallPost`: Fired when a plugin has finished downloading.
- `PluginUpdatePost`: Fired when a plugin has finished updating.

{{< admonition info "Git and Github." >}}
I will be using `git clone` and `git pull` for downloading and updating the
plugin. Although, it can be changed with minimal tweaking.
{{< /admonition >}}

## Configuration
