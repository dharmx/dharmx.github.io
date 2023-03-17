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

I am kind of against the idea of having a configuration `table` for this but having this will be quite nice. For,
example we would be able to change the `packpath` and the `git` commands that are used for cloning and pulling
a plugin. There are many more features that can be added but, we need to be satisfied with just this for now.

```lua
M._defaults = {
  parson_path = vim.fn.stdpath("data") .. "/site/pack/parson",
  repo_site = "https://github.com/%s.git",
  clone = function(link, location)
    return {
      "git",
      "clone",
      "--depth=1",
      "--recurse-submodules",
      "--shallow-submodules", link, location
    }
  end,
  pull = function(location)
    return {
      "git",
      "pull",
      "--recurse-submodules",
      "--update-shallow", location
    }
  end,
}
M._config = M._defaults
```

That's right. The `_defaults` table should be read-only. And, the `_config` table will be the current config values.
The `_defaults` table will be acting as a fallback which will overwrite `_config` if the user applies a configuration
value which turns out to be not valid.

## Database

In the beginning my initial thought was to use getters, setters, appenders, prependers, etc. But, thinking forward
it seemed like the lines of code would helplessly increase if I decided to take that route.
That is why I decided to use a different route which may seem weird for some.

I decided to make use of the metatables instead. For instance. let us take a `table` named `_database` which will be
our database for storing plugins. Then we'd need to set some metatable(s) for that database. And, note that this
`_database` shall be a member of the `M` table. So, this means that when the user uses this plugin manager they would
need to import `parson.lua` file, which (mind you) will return this `M` table, which will contain the `_database`
table. It is up to the user, what they want to do with `_database`. They may add plugins to it, remove plugins
from it, etc.

```lua
-- parson.lua
local M = {}

M._database = {...}
setmetatable(M._database, {...})
-- USAGE:
-- local parson = require("parson")
-- local db = parson._database
-- db["telescope.nvim"] = {
--   "nvim-telescope/telescope.nvim",
--   config = function()
--     require("telescope_config")
--   end,
-- }
-- ... add more plugins

return M
```

### Sample Usage

Now, the above usage example is perfectly fine but it would look a bit cluttered if you have a huge number of plugins.
Hence, we shall change the usage API to addition instead of manually creating a `table` key. We would achieve this
by setting the `__add` metatable (FYI). Therefore, take a look at the following API sample.

```lua
local configs = require("configs")
local parson = require("parson")
local db = parson._db
db = db
  + { "nvim-telescope/telescope.nvim", config = configs.telescope, lazy = true }
  + { "nvim-treesitter/nvim-treesitter", config = configs.tree }
  + { "akinsho/bufferline.nvim" }
  + { "goolord/alpha-nvim" }
  + { "AndrewRadev/linediff.vim", lazy = true }
  + { "tpope/vim-repeat", lazy = true }
vim.pretty_print(db)
```

Okay! Now we need to actually implement the plugin specification. Therefore we shall finally write the `__add`
function now 😅.

{{< admonition info "What we need to implement." >}}
The following is the plugin specification. The user would need to provide the `"head/tail"` of the plugin link and
a few more keys (they are optional after all). Then from that data alone we would need to generate the following `table`.

```lua
["aerial.nvim"] = {
  after_download = "<function 1>",
  after_update = "<function 2>",
  config = "<function 3>",
  download = "<function 4>",
  installed = true,
  lazy = false,
  link = "https://github.com/stevearc/aerial.nvim.git",
  load = "<function 5>",
  loaded = false,
  location = "/home/maker/.local/share/nvim/site/pack/parson/start/aerial.nvim",
  name = "stevearc/aerial.nvim",
  repo = "aerial.nvim",
  update = "<function 6>",
  username = "stevearc"
}
```

{{< /admonition >}}

### Optional Keys

Moving forward, we now need to put aside the specification keys that are optional. Hence, the following keys are
needed to be replaced with a stub.

- `after_download`: This callback will be called right after the `git clone` operation gets finished.
- `after_update`: This callback will be called right after the `git pull` operation gets finished.
- `config`: This function will be called after the plugin package is added with `packadd`.

```lua
-- new is the plugin that is to be added into the parson database
new.config = vim.F.if_nil(new.config, function() end)
new.after_download = vim.F.if_nil(new.after_download, function() end)
new.after_update = vim.F.if_nil(new.after_update, function() end)
```

### Simple Keys

Next, we need to assign values to some simple plugin specification keys. They are mentioned in the following list.

- `lazy`: <samp>[:help packadd](https://vimhelp.org/repeat.txt.html#%3Apackadd)</samp> would explain this better.
  Basically, setting this to `true` will clone the plugin repository into the `opt` directory and setting it to
  `false` will clone it in `start`.
- `link`: The URL that will be used to clone the repository.
- `loaded`: This needs to be set if the plugin has already been added into `packpath`. Additionally, the plugins that are
  marked with `lazy = false`, will be marked as `loaded = true`.
- `location`: Local path of the plugin.
- `repo`: The repository title of the plugin. This is to be derived from `link`.
- `username`: The repository author of the plugin. This is to be derived from `link`.
- `name`: Name of the plugin. This is a combination of `username/repo` which is to be derived from `link`.

```lua
-- take: { "AndrewRadev/linediff.vim", lazy = true }
-- we are removing plugin[1] ∴ name = "AndrewRadev/linediff.vim"
new.name = vim.F.if_nil(new.name, table.remove(new, 1))
new.lazy = vim.F.if_nil(new.lazy, false)
-- assign username = AndrewRadev and repo = linediff.vim
new.username, new.repo = unpack(vim.split(new.name, "/", { plain = true }))
-- since lazy is true then location = /home/username/.local/share/nvim/site/pack/parson/opt/linediff.vim
new.location = string.format("%s/%s/%s", M._config.parson_path, new.lazy and "opt" or "start", new.repo)
new.link = vim.F.if_nil(new.link, string.format(M._config.repo_site, new.name))
new.loaded = false -- defaulted
new.installed = not not vim.loop.fs_realpath(new.location) -- see :help uv.fs_realpath()
```

{{< admonition info "What is this new table?" >}}
The `new` table is an argument of the `__add` meta-method. See the following.

```lua
M._config = M._defaults
M._database = setmetatable({}, {
  __add = function(database, new)
    new.name = vim.F.if_nil(new.name, table.remove(new, 1))
    new.lazy = vim.F.if_nil(new.lazy, false)
    new.username, new.repo = unpack(vim.split(new.name, "/", { plain = true }))
    new.location = string.format("%s/%s/%s", M._config.parson_path, new.lazy and "opt" or "start", new.repo)
    new.link = vim.F.if_nil(new.link, string.format(M._config.repo_site, new.name))
    -- add load function
    -- add download function
    -- add update function
    ...
    M._database[new.repo] = new -- finally add plugin to database
    return database
  end,
})
```

{{< /admonition >}}

## Main Keys

We will be implementing the defaults of the following keys (because they can be overwritten after all).

- `load`: This function should be responsible for adding the plugin directory to the `packpath`.
  Hence, a call to `vim.cmd.packpath(...)` is absolutely necessary.
- `download`: This function is responsible for cloning the plugin repository locally.
- `update`: This function is responsible for updating the plugin repository.

This three keys are the very core of this plugin manager.

### <samp>load</samp>

The loader function. The implementation will be really short. This is because we are leveraging Neovim's builtin
package managing functionality. Which is done adding a directory to `packpath` that contains `start` and `opt`.
This will make Neovim load plugins that are under `start` on startup and will make plugins in `opt` to be loaded
by `:packadd`. Anyway let us assume that those paths are already available foor use.

Now we need to do the following things in the default implementation of `load`.

1. Invoke `vim.cmd.packadd` and pass the `new.repo` key to it.
2. Mark `new.loaded` as `true`.
3. Call the `new.config` function.

Take a look at the following code.

```lua
function new.load()
  vim.cmd.packadd(new.repo)
  new.loaded = true
  new.config()
end
```

### <samp>download</samp>

This one will be a bit longer than `load`. Mainly because we need to write a async task function for this.
This so called "task" will asynchronously run a `git clone` command and after this has finished running
a `on_exit` callback will be called, passing `code`, `signal` and `stdio` pipes to it. See the following.

```lua
local function Task(options)
  local pipes = { stdout = vim.loop.new_pipe(), stderr = vim.loop.new_pipe() }
  -- options.cmd = { git, clone, repo, link }
  -- table.remove(options.cmd, 1) -> git
  -- options.cmd -> { clone, repo, link }
  vim.loop.spawn(table.remove(options.cmd, 1), {
    args = #options.cmd == 0 and nil or options.cmd,
    cwd = options.directory,
    stdio = vim.tbl_values(pipes),
  }, function(code, signal)
    if options.on_exit then options.on_exit(code, signal, pipes) end
  end)
end
```

Next we need to implement `download`. And similar to `load`, we need to mark `new.installed = true` after
the clone task completes successfully.

```lua
function new.download()
  local config = { cmd = M._config.clone(new.link) }
  function config.on_exit(code, _, pipes)
    if code == 0 then new.installed = true end
    new.after_download(new, pipes)
  end
  -- ~/.local/share/nvim/site/pack/parson/start/alpha-nvim
  -- -> ~/.local/share/nvim/site/pack/parson/start
  config.directory = vim.fn.fnamemodify(new.location, ":h")
  -- git clone repo will be done in ~/.local/share/nvim/site/pack/parson/start
  -- this will create a repo directory in ~/.local/share/nvim/site/pack/parson/start
  -- -> ~/.local/share/nvim/site/pack/parson/start/repo i.e. same as new.location
  Task(config)
end
```

### <samp>update</samp>

This will mostly be the same as `download`.

```lua
function new.update()
  local config = { cmd = M._config.pull() }
  function config.on_exit(code, _, pipes)
    if code == 0 then new.installed = true end
    new.after_update(new, pipes)
  end
  -- cd to new.location and then run git pull
  config.directory = new.location
  Task(config)
end
```

## Setup

### Full Code

### Test Usage

## Lazy Handlers
