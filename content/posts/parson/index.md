---
draft: true

title: "Parson"
subtitle: "A minimal package manager for Neovim. Sample configuration included."
summary: "Decided to write a package manager for Neovim within 100 lines of code.
          Turned out pretty well in my opinion. Additionally, features that could
          not be included are also documented."
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

images: ["/parson/images/featured-image.png"]
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

Hello again. It's been a while since my wrote my last post. Been really busy with studies and whatnot. Anyways! This
post is going to be a bit different. It will be a exploration and experimentation kind of post i.e. I will attempt
to create a minimal plugin manager for Neovim. It should be under 100 lines of code.

{{< admonition warning "Code Quality." false >}}
Please, keep in mind that the code may end up looking really cluttered. This is because I am optimizing towards
lines of code. So, I'll probably end up squeezing as much stuff as humanely possible. Although, you are allowed
to critique me on parts that could be shortened. Well... other suggestions are allowed too! But, please be
courteous when you are doing so. (**BE NICE**)
{{< /admonition >}}

## Plan

Currently, I am planning to add the following features.

- Define a nice API for adding plugin specifications.
- Allow downloading to `opt` to support Vim's default lazy loading mechanism.
- Implement `update`, `download`, `config` and `load` functionalities.
- Allow `config`, `after_download` and `after_update` callbacks.
- Store `installed`, `location`, `link`, `loaded`, `name`, and `username` metadata keys.

Additionally, I will be implementing more advanced lazy handlers but, won't be adding them to the plugin code.
Instead, I will add an "extras" section and add code snippets with minimal descriptions there. Lastly, I will
also attach a minimal nvim configuration file (init.lua) that utilizes Parson.

{{< admonition info "Before we proceed." >}}
This is a fairly advanced Neovim/Lua article. Therefore, moderate familiarity with the Lua programming language
is required. Additionally, some familiarity with Neovim Lua API is highly preferred. I will also simplify what each
Neovim API does along the way as best as I can.
{{< /admonition >}}

## Metatables

I will mainly be explaining how the `__add`, `__sub`, `__index`, `__call` and `__newindex` metatables work. Out
of this the `__add` metatable will be the backbone of Parson so pay extra attention! Anyway, for starters,
just know that metatables allow us to change the behaviour of a table. It does so by using specific events
i.e. say you want to add two items `a + b` where `a` and `b` are tables. Whenever Lua tries to add two tables,
it checks whether either of them has a metatable and whether that metatable has an `__add` field.
If Lua finds this field, it calls the corresponding value (the so-called metamethod, which should be a function)
to compute the sum. If you have enough experience with metatables then you may skip to the
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
vim.pretty_print(MANGA) -- run by :luafile %
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
vim.pretty_print(MANGA)
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

This one's a lot easier than the ones we have seen so far. This metamethod runs when you try to index a `nil`
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
vim.pretty_print(_EXTRA[2])
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

This is similar to the `__index` metatable. This metamethod runs when you try to index and set a `nil` element of the
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
vim.pretty_print(MANGA("hello.com"))
vim.pretty_print(MANGA({ "garbage.com", "wuxiaworld.com", "gogomanga.com" }))
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

### vim.o and vim.opt

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
vim.pretty_print(vim.opt.fillchars) -- NOTE: This will print a table instead.

-- these are a bit scuffed
vim.pretty_print(vim.opt.fillchars._value) -- same
vim.pretty_print(vim.opt.fillchars:get()) -- same

-- INFO: If all you want is to inspect the current value of a setting then you may prefer vim.o instead.
vim.pretty_print(vim.o.fillchars) -- same as vim.opt.fillchars:get()
```

I hope the difference is clear now.

### packages and packpath

Please read the <samp>[runtimepath](https://vimhelp.org/options.txt.html#%27runtimepath%27)</samp> help page section.
Then the <samp>[packpath](https://vimhelp.org/options.txt.html#%27packpath%27)</samp> section and finally, the
<samp>[packages](https://vimhelp.org/repeat.txt.html#packages)</samp> section. I am too lazy to explain this. I'll
edit and write my own explanations and examples when I'm in mood. Let us quickly move to the main part 🤩!
I am fed up with distractions!

## Plugin Specification

You are entering the core of this article! Make sure to stay hydrated 🛫.

For starters, please skim through this [readme](https://github.com/nvim-lua/nvim-package-specification) this is an
unofficial draft but it is useful for getting a gist of what this section will be about. Moving on, we will only
take/adapt the `package` and `source` from this. And, some of my custom keys will also be added! Hence, please take a
look at the brief description below.

- `name`: The name of the plugin. This will be an adaption of the `package` key from
  [readme](https://github.com/nvim-lua/nvim-package-specification). The main difference is that
  the name will be the tail of the GitHub repository rather than the namespace i.e. we will be using
  `https://github.com/neovim/nvim-lspconfig →  nvim-lspconfig` as opposed to `nvim-lspconfig/lua/lspconfig →  lspconfig`.
- `source`: This will be similar to `source` but we'll alias it as `link` (this is because I have already written the
  prototype which this article is all about 😅).
- `username`: We will be splitting the `https://github.com/neovim/nvim-lspconfig` link with respect to `/` and will
  take the parent of `nvim-lspconfig` i.e. `neovim`. This is basically a indicator of the author of the plugin.
  This is useless most of the time but with given data (the link) it is effortless to generate.
- `repo`: The naming is not the best for this one (maybe `repo_title` would be better?). This is basically the
  tail of the link.
- `location`: The `repo` key will be useful here. This is basically the local location of the plugin.
- `installed`: A key that will be set by the plugin itself after the plugin is downloaded to the `location` path.
  Additionally, the plugin manager will also check if the plugin is already installed and set that field accordingly.
- `loaded`: This will be similar to `loaded`. The difference: It will be set to `true` after
  <samp>[:packadd](https://neovim.io/doc/user/repeat.html#%3Apackadd)</samp>.
- `lazy`: This will decide whether the plugin will be manually loaded or, will be loaded on [startup](https://vimhelp.org/starting.txt.html#startup).

There are more keys. Which I have decided to separate them from the main list. This list contains callbacks (optional).
I was contemplating if I should remove these for making the code more minimal. But now that I think about it, it seems
that these sorts of callbacks (`on_action`, `before_action`, etc) will also server as a nice illustration.

- `after_download`: These are optional callbacks that will be called after **a** plugin finishes downloading.
- `after_update`: Same as `after_download` except this one is called after plugin update.

Yet another batch of specification keys. These are also functions but they are not optional. But, overriding them is.
This is the last batch so rest assured 😄.

- `config`: This function will be responsible for loading the plugin configuration. This needs to run after loading
  the plugin (plugins are loaded by the <samp>:packadd</samp> command).
- `load`: This function will be responsible for loading the actual plugin.
- `download`: This function will be responsible for asynchronously downloading the plugin package into a local
  directory.
- `update`: This function will be responsible for asynchronously updating the plugin directory.

{{< admonition info "Git and Github." >}}
I will be using `git` and [GitHub](https://github.com) for `git clone` and `git pull` for downloading and updating the
plugin. Although it can be changed with minimal tweaking.
{{< /admonition >}}

## Configuration

## Database

### Sample Usage

### Optional Keys

### Simple Keys

## Main Keys

## Setup

### Full Code

### Test Usage

## Lazy Handlers
