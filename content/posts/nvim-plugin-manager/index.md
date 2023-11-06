---
draft: true

title: "Neovim Plugin Manager"
subtitle: "A bare-bones package manager for Neovim. Sample configuration included."
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

Hello again. It's been quite a while since my wrote my last post. Been really busy with studies
and some entrance examinations.
This time I will be writing code for once. So, we will be creating a nice and simple plugin manager for Neovim.
This will help in getting a deeper and clearer understaning of how plugin managers actually work in Neovim.

{{< admonition warning "Code Quality." false >}}
Please, keep in mind that the code may end up looking really cluttered. This is because I am optimizing towards
lines of code. So, I'll probably end up squeezing as much contents as humanely possible.
{{< /admonition >}}

## Plan

Currently, I am planning to add the following features.

- Define a nice API for adding plugin specifications.
- Allow downloading to `opt` to support Vim's default lazy loading mechanism.
- Allow `PluginLoadPost`, `PluginInstallPost` and `PluginUpdatePost` auto-commands.
- Store `installed`, `location`, `link`, `loaded`, `name`, and `username` metadata keys.

Additionally, I will be implementing more advanced lazy handlers but won't be adding them to the plugin code.
Instead I will be adding an "Extras" section and co-responding code snippets with minimal descriptions there.
Lastly, I will also attach a minimal nvim configuration file for a demonstration.

{{< admonition info "Before we proceed." >}}
This is a fairly advanced Neovim/Lua article. Therefore, moderate familiarity with the Lua programming language
is required. Additionally, some familiarity with Neovim Lua API is preferred. I will also simplify what each
Neovim API does along the way as best as I can.
{{< /admonition >}}

## Specification

For starters, please skim through this
[nvim-package-specification](https://github.com/nvim-lua/nvim-package-specification) document.
This is an unofficial draft but it is useful for getting a gist of what this section will be about.
We will only _adapt_ the `package` and `source` from this. And, some of custom keys will also be added!

- `[1]`: The source of the plugin. The main difference from the draft is that this will use
  [URI](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier)
  And, it will effectively combine the `name` and the `url` keys from the draft.
- `opt`: This will decide whether a plugin will be downloaded/linked to the `opt` directory or, the
  `start` directory. Learn more about `opt` and `start` directories [here](https://neovim.io/doc/user/repeat.html#packages).

## Events

Most plugin managers have plugin specification keys like `config`, `build`, `dependencies`, etc.
But, we won't be implementing them strictly here. Instead, we will try to leverage auto-commands
as much as possible.

The core concept is for example, fire an user event when a plugin is loaded. We can then bind a
callback to that event which will check which plugin has been loaded based on the metadata passed
into the callback and then load its configuration. For making this concerte we will implement the
following events.

- `PluginBeforeApply`: Fired before a plugin has been parsed.
- `PluginAfterApply`: Fired after a plugin has been parsed.
- `PluginAfterFetch`: Fired when a plugin has finished downloading.

{{< admonition info "Git" >}}
I will be using `git clone` for downloading the plugin.
Although, this can be changed with minimal tweaking.
{{< /admonition >}}

## Configuration

We will be using a dummy configuration for starters. Then build the plugin
manager in accordance to the configuration. Thus, take a look at the following
code.

```lua
-- store this in a variable to save the parsed plugin specifications
require("npm") {
  { "https://github.com/dharmx/colo.nvim.git"                         },
  { "file://home/dharmx/Dotfiles/neovim/track.nvim"     , opt = false },
  { "https://github.com/dharmx/telescope-media.nvim.git", opt = true  },
}
```

From the above code, we can infer that requiring `"npm"` will return a callable
`table` which when called it will return itself this is done to enable chaining.

We will be using [URIs](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier) for adding plugins.
For example, if the URI is of [HTTP(S)](https://en.wikipedia.org/wiki/HTTPS)
scheme then it is assumed to be a link to a [GIT](https://en.wikipedia.org/wiki/Git)
repository. And, if it is of [FILE](https://en.wikipedia.org/wiki/File_URI_scheme)
scheme then it should be assumed to be a link to a local plugin.

See the [specification]({{< ref "#specification" >}}) section for knowing what `opt = false` means.

## NPM

Based naming sense :sunglasses:.

Alrighty. For starters, let us claim some namespaces :triangular_flag:.

- `$HOME/.cache/nvim/npm`: Used for storing logs.
- `$HOME/.local/share/nvim/npm`: Used for storing plugins.
- `$HOME/.local/state/nvim/npm`: Used for storing lockfiles and other states.
- `$HOME/.local/share/nvim/npm/opt`: Used for storing plugins that are to be lazy loaded.
- `$HOME/.local/share/nvim/npm/start`: Used for storing plugins that are to be loaded immediately on startup.

Store that in a `table`. Only the ones we need at the moment. Then we shall add the
plugin store namespaces into `packpath`. Note that we would need to create these
directories if they do not exist.

```lua
local pack = {
  site = vim.fn.stdpath("data") .. "/site",
  head = vim.fn.stdpath("data") .. "/site/pack/npm",
  start = vim.fn.stdpath("data") .. "/site/pack/npm/start", -- non-lazy plugin location
  opt = vim.fn.stdpath("data") .. "/site/pack/npm/opt", -- lazy plugin location
}

-- add plugin download directory to packpath
-- this will allow :packadd if a plugin is lazy
vim.opt.packpath:prepend(pack.site)
if vim.fn.isdirectory(pack.head) == 0 then
  vim.fn.mkdir(pack.start, "p")
  vim.fn.mkdir(pack.opt)
end
```

Now, we just need to return a `table` with the `__call` meta-method defined. The `_call`
meta-method at the moment will just go through the list of `plugins` that is passed
into it and then delegate it to a parser function i.e. `apply` as in apply specification
and then it will append the plugin entry to itself.

```lua
-- appended after the previous snippet
return setmetatable({}, {
  __call = function(self, plugins)
    for index, plugin in ipairs(plugins) do
      apply(plugin)
      rawset(self, index, plugin)
    end
    return self
  end,
})
```

### Applying Specification

Now, we define the `apply` function. This will be really minimal.
We only need it to download and put said downloaded things in the right place.
Now, take a look at the implementation.

```lua
-- the previous snippet is to be put here
local function apply(plugin)
  local URI = plugin[1]
  local startopt = plugin.opt and pack.opt or pack.start
  local location = string.format("%s/%s", startopt, vim.fn.fnamemodify(URI, ":t"))
  if URI_type == "http" or URI_type == "https" then
    -- if plugin directory does not exist then download the plugin
    if (vim.loop.fs_realpath(location)) then
      vim.fn.jobstart(string.format("git clone --depth=1 --recurse-submodules %s %s", URI, location))
    end
  else error(URI .. " URI-type is not supported") end -- not using http(s) URIs will result in an error
  plugin.location = location
end
```

1. First of all we are constructing a local path from the plugin URI. For instance, if we
   have a URI like `https://github.com/dharmx/toml.nvim` then we extract the tail from it i.e. `toml.nvim`
   And, based off of the `plugin.opt` value we determine whether it will go into `$HOME/.local/share/nvim/npm/opt`
   or, `$HOME/.local/share/nvim/npm/start`.
2. Then we are validating the URI-type i.e. for now we must only allow either HTTP or, HTTPS links.
3. Now, if the constructed location already exists then we skip it, if not then we download it.

And, that's it! Congratulations, on making it to the end :smile:. You now have your very own DIY plugin manager.

From this point on, we delve into enhancements. You may close this if you are not interested.

### Custom Directories

As the title implies, we now need to support the `file://` URI scheme. We will be doing this by just
adding a branch case at the place where the URI-type is being checked. Additionally, we need to assert
that the local path exists. And, if it does, then we shall symlink into either the `opt` or, the `start`
directory which is based on the `plugin.opt = true` option.

```lua
...
local function apply(plugin)
  local URI = plugin[1]
  -- checks only for http(s) and file URIs
  local URI_type = assert(vim.trim(URI):match("^(%w+)://.+"), URI .. " invalid URI")
  local startopt = plugin.opt and pack.opt or pack.start
  local location = string.format("%s/%s", startopt, vim.fn.fnamemodify(URI, ":t"))
  if URI_type == "file" then
    -- download and symlink if URI is offline based
    local real_location = vim.uri_to_fname(URI)
    assert((vim.loop.fs_realpath(real_location)), URI .. " not found")
    vim.loop.fs_symlink(real_location, location) -- offline/local plugins are to be symlinked
  elseif URI_type == "http" or URI_type == "https" then
    if (vim.loop.fs_realpath(location)) then -- if plugin directory does not exist then download plugin
      vim.fn.jobstart(string.format("git clone --depth=1 --recurse-submodules %s %s", URI, location))
    end
  else error(URI .. " URI-type is not supported") end -- not using http(s)/file URIs will throw errors
  plugin.location = location
end
```

## Update

Alright. This should be pretty straight-forward. We just need to `pull` the changes
from all `http(s)` plugins. This will be similar to what the previous section did
i.e. we will have a asynchronous job that will run several `git pull` commands.

```lua
local function update(plugin)
  local URI = plugin[1]
  local URI_type = assert(vim.trim(URI):match("^(%w+)://.+"), URI .. " invalid URI")
  local location = plugin.location
  if URI_type == "http" or URI_type == "https" then
    if vim.loop.fs_realpath(location) == nil then -- if plugin directory ~exist ∴ download plugin
      install(plugin)
    else -- otherwise initiate git pull
      vim.fn.jobstart("git pull --no-recurse-submodules --no-rebase", { cwd = location })
    end
  end
end
```

## Clean

<!-- remove orphaned plugins and disabled plugins -->

## Firing Events

Like [packer.nvim](https://github.com/wbthomason/packer.nvim),
[lazy.nvim](https://github.com/folke/lazy.nvim), [vim-plug](https://github.com/junegunn/vim-plug), etc.
We need to have some sort of events for certain conditions which will allow
an user to do more with the plugin. As, an example: suppose we have a plugin
titled `dharmx/toml.nvim` which just happens to use a command line utility
called `toml2json` which needs to be built every time a plugin receives updates.
Now, to determine when a plugin is being updated, we would either need to
implement a hook in the plugin manager that gets called _after_ an update or,
we fire an event when the plugin gets updated which will allow the users to
subscribe to an event as many times as they need.

By events I mean auto-commands i.e. we will be firing user events for specific
actions. Hooks are also a great idea but we are essentially reinventing the wheel
in this case. Why do that when a nice construct already exists and which is
vim-like. Whenever someone mentions events in the context of vim they always
mean auto-commands. Therefore, we will be using auto-commands.
We will be using [nvim_exec_autocmds()](https://neovim.io/doc/user/api.html#nvim_exec_autocmds())
instead of [:doautocmd](https://neovim.io/doc/user/autocmd.html#autocmd-execute)
in this case.

Moving on. We will be firing three events (for now).

1. When a plugin has finished cloning. Similar to `do` from vim-plug and `build` from lazy.nvim.
2. Before plugin specifications has been applied. This is similar to lazy.nvim's `init` option.
3. After plugin specifications has been applied.

Therefore, rewriting the `apply` and `return` value, we will have the following.

```lua
...
local function apply(plugin)
  local URI = plugin[1]
  -- checks only for http(s) and file URIs
  local URI_type = assert(vim.trim(URI):match("^(%w+)://.+"), URI .. " invalid URI")
  local startopt = plugin.opt and pack.opt or pack.start
  local location = string.format("%s/%s", startopt, vim.fn.fnamemodify(URI, ":t"))
  if URI_type == "file" then
    -- download and symlink if URI is offline based
    local real_location = vim.uri_to_fname(URI)
    assert((vim.loop.fs_realpath(real_location)), URI .. " not found")
    vim.loop.fs_symlink(real_location, location) -- offline/local plugins are to be symlinked
  elseif URI_type == "http" or URI_type == "https" then
    if (vim.loop.fs_realpath(location)) then -- if plugin directory does not exist then download plugin
      vim.fn.jobstart(string.format("git clone --depth=1 --recurse-submodules %s %s", URI, location), {
        on_exit = function(_, code, action)
          vim.api.nvim_exec_autocmds("User", {
            pattern = "PluginAfterFetch",
            data = { action, code, plugin[1] },
          })
        end,
      })
    end
  else error(URI .. " URI-type is not supported") end -- not using http(s)/file URIs will throw errors
  plugin.location = location
end

return setmetatable({}, {
  __call = function(self, plugins)
    for index, plugin in ipairs(plugins) do
      vim.api.nvim_exec_autocmds("User", { pattern = "PluginBeforeApply", data = plugin[1] })
      apply(plugin)
      vim.api.nvim_exec_autocmds("User", { pattern = "PluginAfterApply", data = plugin[1] })
      rawset(self, index, plugin)
    end
    return self
  end,
})
```

Refer to [nvim_create_autocmd()](https://neovim.io/doc/user/api.html#nvim_create_autocmd())
page before we proceed. You may need to scroll to find what arguments are passed into the
auto-command callback. We would need the args.data key in order to determine what plugin has
been currently downloaded (say). We will subscribe and do some extra things to it.
See the following illustration.

```lua
-- subscribe to plugin events
vim.api.nvim_create_autocmd("User", {
  pattern = { "PluginBeforeApply", "PluginAfterApply", "PluginAfterFetch" },
  callback = function(args) vim.notify(vim.inspect(args)) end,
})

-- show warning dialog if args.data contains a string that ends with "track.nvim"
vim.api.nvim_create_autocmd("User", {
  pattern = "PluginAfterFetch",
  callback = function(args)
    local plugin = args.data
    if plugin:match("track.nvim$") then
      vim.notify("This is a WIP plugin.")
    end
  end,
})
```

This is it for now. I will be adding more events and illustrating their usages as
more features are added to NPM.

## Reorganize

<!-- keep pin, disable, commit, tag and dev in mind. -->

### Returns

Rather than littering the returned table with first level plugin entries, we will be
creating a `plugins` entry and push all of those plugin entries into this entry.
Therefore please take a look at this updated return value.

```lua
return setmetatable({
  plugins = {},
  state = {},
  core = {},
}, {
  plugins = {},
  __call = function(self, plugins)
    local store = rawget(self, "plugins")
    for index, plugin in ipairs(plugins) do
      apply(plugin)
      table.insert(store, index, plugin)
    end
    rawset(self, "plugins", store)
    return self
  end,
})
```

{{< admonition question "Why is this change necessary?" false >}}
It would be difficult to parse or, filter things that are not plugin entries.
For instance, what if there are API functions and state variables?
We could filter them out but that is an extra step.
Therefore, we seperate them into three nested tables.

```lua
-- before: mixed categories
local NPM = {
  -- plugins
  -- variables
  -- API functions
}

-- after: organized categories
local NPM = {
  plugins = {...}, -- plugin entries
  state = {...}, -- variables
  core = {...}, -- API functions
}
```

Moving forward, we can now include all API functions like `install`, `sync`, `update`, `show`,
etc inside the `NPM.core` namespace.
{{< /admonition >}}

### Fragmenting Apply

We will be move-out a part of the `apply()` to the newly created `install()` function.

```lua
...
local function install(plugin)
  local URI = plugin[1]
  -- checks only for http(s) and file URIs
  local URI_type = assert(vim.trim(URI):match("^(%w+)://.+"), URI .. " invalid URI")
  local location = plugin.location
  if URI_type == "file" then
    -- download and symlink if URI is offline based
    local real_location = vim.uri_to_fname(URI)
    assert(vim.loop.fs_realpath(real_location) ~= nil, URI .. " not found")
    vim.loop.fs_symlink(real_location, location) -- offline/local plugins are to be symlinked
  elseif URI_type == "http" or URI_type == "https" then
    if vim.loop.fs_realpath(location) == nil then -- if plugin directory does not exist then download the plugin
      vim.fn.jobstart(string.format("git clone --depth=1 --recurse-submodules %s %s", URI, location), {
        on_exit = function(_, code, action)
          vim.api.nvim_exec_autocmds("User", { pattern = "PluginAfterFetch", data = { action, code, plugin[1] } })
        end,
      })
    end
  else error(URI .. " URI-type is not supported") end -- not using http(s)/file URIs will result in an error
  plugin.location = location
end

local function apply(plugin)
  plugin.location = string.format("%s/%s", lazy(plugin) and pack.opt or pack.start, vim.fn.fnamemodify(plugin[1], ":t"))
  install(plugin)
end
...
```

## Module

<!-- turn the file into a lua module by using M -->

## Loaders

Loaders are mechanisms that will load a plugin. We in this section, will look specifically
at variants of loaders i.e. lazy-loaders. Now, as the name suggests "lazy-loaders" will
load said plugin _only_ when needed. This is done in order to make startup times faster and
the UI more responsive, snappy, etc. If one has 150+ plugins then it is more than likely that
their startup times will be hampered and Neovim will be much less snappy without proper
lazy-loading.

In this section we will be first implement some fairly simple loaders like the following.

- `ev`: Load plugin(s) on a specific auto-command event.
- `ft`: Load plugin(s) on a specific filetype.
- `time`: Load a plugin after a specific time.

### Event

This will be fairly simple. We just need to create a auto-group first then create auto-commands
for each plugin specification. Therefore, take a look at the following code.

```lua
...
local group = vim.api.nvim_create_augroup("NPM", { clear = false })

local function lazy(plugin)
  return not not (plugin.opt or plugin.ev or plugin.ft or plugin.time)
end

local function apply_events(plugin)
  if not plugin.ev then return end
  vim.api.nvim_create_autocmd(vim.tbl_flatten({ plugin.ev }), {
    once = true,
    group = group,
    command = "packadd " .. vim.fn.fnamemodify(plugin.location, ":t"),
  })
end

...

local function apply(plugin)
  plugin.location = string.format("%s/%s", lazy(plugin) and pack.opt or pack.start, vim.fn.fnamemodify(plugin[1], ":t"))
  install(plugin)
  apply_events(plugin)
end
...
```

We will allow both `string` and `table` for assigning plugin events. This is the reason you
will notice that the code is using `tbl_flatten({ plugin.ev })`. And, the event handler
is to be executed only once since we cannot load the same plugin twice. And, another
thing to note is that we also defined a `lazy` function. This is done because from this
point on there will be some new specification keys that will imply that the plugin needs
to be lazy-loaded like the newly added `ev` and others like `ft`, `time`, etc.

Before we move on, take a look at the newly updated configuration.

```lua
local npm = require("npm") {
  { "file://home/dharmx/Dotfiles/neovim/track.nvim" , ev  = "CursorMoved" },
  { "https://github.com/dharmx/telescope-media.nvim", opt = true          },
  { "https://github.com/dharmx/colo.nvim"           , opt = true          },
}
```

### FileType

This will be similar to `ev` i.e. we need to create auto-commands which will subscribe to
a specific event called `FileType`. Then we need to check whether the file type of the
current buffer is same as the one specified in the plugin specification. Therefore,
consider the following configuration.

```lua
require("npm") {
  { "file://home/dharmx/Dotfiles/neovim/track.nvim" , ev  = "CursorMoved" },
  { "https://github.com/dharmx/telescope-media.nvim", ft  = "lua"         },
  { "https://github.com/dharmx/colo.nvim"           , opt = true          },
}
```

Now, based on this configuration, we will be adjusting our plugin manager.

```lua
...
local function apply_filetype(plugin)
  if not plugin.ft then return end
  vim.api.nvim_create_autocmd("FileType", {
    once = true,
    group = group,
    callback = function()
      if vim.tbl_contains(vim.tbl_flatten({ plugin.ft }), vim.bo.filetype) then
        vim.cmd.packadd(vim.fn.fnamemodify(plugin.location, ":t"))
      end
    end,
  })
end

...

local function apply(plugin)
  plugin.location = string.format("%s/%s", lazy(plugin) and pack.opt or pack.start, vim.fn.fnamemodify(plugin[1], ":t"))
  install(plugin)
  apply_events(plugin)
  apply_filetype(plugin)
end
...
```

### Time

Delayed load. This will load plugins after a specific amount of time. This specification
will take in milliseconds. For example, if `3000` milliseconds is specified then the plugin
will load three seconds _after_ when the specification is applied. You can assume it being
roughly the time when `VimEnter` and `UIEnter` events are fired.

See the following configuration.

```lua
require("npm") {
  { "file://home/dharmx/Dotfiles/neovim/track.nvim" , ev   = "CursorMoved" },
  { "https://github.com/dharmx/telescope-media.nvim", ft   = "lua"         },
  { "https://github.com/dharmx/toml.nvim"           , opt  = true          },
  { "https://github.com/dharmx/colo.nvim"           , time = 10000         },
}
```

And, the specification function goes as follows.

```lua
...
local function apply_time(plugin)
  if not plugin.time then return end
  ---@see https://neovim.io/doc/user/luvref.html#luv-timer-handle
  vim.loop.new_timer():start(plugin.time, 0, vim.schedule_wrap(function()
    vim.cmd.packadd(vim.fn.fnamemodify(plugin.location, ":t"))
  end))
end

...

local function apply(plugin)
  plugin.location = string.format("%s/%s", lazy(plugin) and pack.opt or pack.start, vim.fn.fnamemodify(plugin[1], ":t"))
  install(plugin)
  apply_events(plugin)
  apply_filetype(plugin)
  apply_time(plugin)
end
...
```

Here, we are creating a [uv_timer_t](http://docs.libuv.org/en/v1.x/timer.html) object
by using the [libuv](https://libuv.org) library, which can be accessed through `vim.loop`.
After creating the timer, we immediately start. We are setting the interval
parameter to `0` as we only need it to run once and not repeat.

## Oneshots

Small and easy-to-implement specification keys. These are instantaneous. For example,
loading a plugin based on a boolean expression i.e. `exists(some_legacy_function)`.
Another example could be downloading a plugin and then locking is version which will
make it invisible to updates.

Please, take a moment to examine the following specification keys.

- `cond`: A boolean expression or, a function which will return a boolean value.
- `pin`: Make a plugin invisible to updates i.e. maintain current version indefinitely.
- `disable`: Mark plugin for removal.
- `dev`: Allow switching between `plugin[1]:https` and `plugin[2]:file` using booleans.

Moving forward, we need to update our `lazy()` utility function first.

```lua
local function lazy(plugin)
  local options = {
    "opt",
    "ev",
    "ft",
    "time",
    "disable",
    "cond",
  }
  local final = false
  for _, option in ipairs(options) do
    final = final or not not plugin[option]
    if final then return final end
  end
  return final
end
```

## VCS

- `submodules`
- `commit`
- `tag`

## Extras

- `config`:
- `build`:
- `after`:
- `deps`:
- `name`:

## Lockfile

## Loop

We NEED to use `vim.loop` Jesse :angry:.

But, yeah.. we are going to remove all usages of `vim.fn` and replace it with
[libuv](https://libuv.org) alternatives. It seems that `vim.fn` is much slower than the
low level libuv equivalents. And for some fucking reason you need to write more!
I guess the APIs covers too many grounds. But it shouldn't matter regardless!
Just abstract it and give us similar functions like `vim.fn.isdirectory`.

There IS [plenary.nvim](https://github.com/nvim-lua/plenary.nvim) which does have `path.lua`
but, it is not available during startup (this is because it is a plugin and it is
mostly used by plugins).

## Showcase

<!-- paste the final code and a GIF demo -->

## Conclusion

<!-- mention writing an UI for NPM -->

## References

- <https://www.lua.org/manual/5.1/index.html#index>
- <https://www.lua.org/manual/5.1/manual.html>
- <http://lua-users.org/wiki/BinaryModulesLoader>
- <http://lua-users.org/wiki/LuaModulesLoader>
- <https://vimhelp.org>
- <https://neovim.io/doc/user/api.html>
- <https://github.com/savq/paq-nvim>
- <https://github.com/folke/lazy.nvim>
- <https://github.com/wbthomason/packer.nvim>
- <https://github.com/junegunn/vim-plug>

## Tools

- [Canva](https://www.canva.com/)
- [GIMP](https://www.gimp.org/)
- [ImageMagick](https://imagemagick.org/)
- [Photopea](https://www.photopea.com/)

## Ending Note

This is the first time I enjoyed writing an entire 4000+ words **all the way**. The
previous articles were nice and all.. but I did not enjoy writing them this much!
Anyways, from this point on you should expect more Neovim related articles.
And, if you want to see something specific please write an Email to me or, write
about it in the comments section down below.

And lastly, any corrections, criticisms and additions are very welcomed! You can
also suggest changes by clicking on the **"Edit this page"** and **"Report issue"**
buttons down below. Cheers :wave:!
