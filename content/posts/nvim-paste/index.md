---
draft: false

title: "Neovim Pastebin"
subtitle: "Small neovim pastebin utility."
summary: "A small minimal utility that posts a neovim buffer to a pastebin website and the copy the returned link into a register."
description: "A pastebin script for sharing code."

date: "2024-03-06T17:49:05+05:30"
lastmod: "2024-03-06T17:48:33+05:30"

tags: ["nvim", "plugin", "scripting"]
categories: ["nvim"]
relative: true

author: "dharmx"
authorLink: "https://dharmx.is-a.dev"
license: "<a rel='license external nofollow noopener noreffer' href='https://opensource.org/licenses/GPL-3.0' target='_blank'>GPL-3.0</a>"

featuredImage: "images/featured-image.png"
featuredImagePreview: "images/featured-image.png"

images: ["/nvim-paste/images/featured-image.png"]
toc:
  auto: true

hiddenFromHomePage: false
hiddenFromSearch: false
twemoji: false
lightgallery: true
ruby: true
fraction: true
fontawesome: true
linkToMarkdown: true
rssFullText: true
---

# Introduction

Recently, I have been sharing a lot code snippets and other textual :smirk: files over the
internet. So, currently I just run a [curl](https://curl.se/) command and then upload
the file to https://0x0.st.

The command looks like this: `curl -F'file=@/absolute/path/to/file' 0x0.st | xclip`.
We will be just transforming this command into a Neovim utility.

{{< admonition info "Other similar plugins." false >}}<!--{{{-->
So, before writing this I did try several other plugins. I did not like any but, maybe
you will.

- There is [paperplanes.nvim](https://github.com/rktjmp/paperplanes.nvim) but it is
  a bit too feature-rich. It supports multiple backends and other fringe features that
  I do not really need.
- There is [pastebin-vim](https://github.com/mattn/pastebin-vim) which is also cumbersome
  to use as you'd need to generate API keys first.
- Then there's [snips.nvim](https://github.com/Sanix-Darker/snips.nvim) which is the best
  so far. But again I want something minimal.
- Lastly, there is [vim-pastebins](https://github.com/sefeng211/vim-pastebins/) which I have not tried.
{{< /admonition >}}<!--}}}-->

## Plenary

We will be using the well-known [plenary.nvim](https://github.com/nvim-lua/plenary.nvim)
library plugin. The only utilities we need from this are the `Job` and `Path` class.
Which just basically [spawns](https://neovim.io/doc/user/luvref.html#uv.spawn())
an async task in the background and calls an `on_exit` callback function when
the task is done. And, the other is just the Lua version of Python's
[pathlib](https://realpython.com/python-pathlib/).

The readme has examples of `Job` but here is an example anyway.

```lua
local Job = require("plenary.job")
Job:new({
  command = "ls"
  args = { "--recursive" },
  on_exit = vim.schedule_wrap(function(self, code, _)
    if code ~= 0 then
      vim.notify(vim.inspect(self:stderr_result()))
      return
    end
    vim.notify(vim.inspect(self:result()))
  end),
})
```
{{< admonition info "Alternative APIs." false >}}<!--{{{-->
Following are alternative APIs to plenary's `Job` :skull:.

- [<samp>:help vim.system()</samp>](https://neovim.io/doc/user/lua.html#vim.system())
- [<samp>:help uv.spawn()</samp>](https://neovim.io/doc/user/luvref.html#uv.spawn())
- [<samp>:help job-control</samp>](https://neovim.io/doc/user/job_control.html)

Following are alternative APIs to plenary's `Path` :cry:.

- [<samp>:help exists()</samp>](https://vimhelp.org/builtin.txt.html#exists%28%29)
- [<samp>:help :glob()</samp>](https://vimhelp.org/builtin.txt.html#glob%28%29)
- [<samp>:help readfile()</samp>](https://vimhelp.org/builtin.txt.html#readfile%28%29)
- [<samp>:help writefile()</samp>](https://vimhelp.org/builtin.txt.html#writefile%28%29)
- [<samp>:help delete()</samp>](https://vimhelp.org/builtin.txt.html#delete%28%29)
- [<samp>:help uv.fs_unlink()</samp>](https://neovim.io/doc/user/luvref.html#uv.fs_unlink()): Delete file.
- [<samp>:help uv.fs_realpath()</samp>](https://neovim.io/doc/user/luvref.html#uv.fs_realpath()): Existence of a path.
- [<samp>:help uv.fs_read()</samp>](https://neovim.io/doc/user/luvref.html#uv.fs_read())
- [<samp>:help uv.fs_write()</samp>](https://neovim.io/doc/user/luvref.html#uv.fs_write())
- [<samp>:help uv.fs_open()</samp>](https://neovim.io/doc/user/luvref.html#uv.fs_open())

A nice little exercise would be to create your version of this script but by using
these alternative APIs instead.
{{< /admonition >}}<!--}}}-->

## Procedure

Here's a list of features that I primarily want.

- Ability to upload a file.
- Ability to upload specific lines from a file.
- Ability to remove the uploaded item.
- Ability to cache uploaded file into a JSON file.

As for going about implementing the upload feature for selected text, we would need to copy
that selected text or, file and paste those contents into another temporary file and _then_
upload that file to the pastebin website.

### Configuration

We just need defaults for paths beforehand. We would need the following paths.

- `db_path`: Path where all previously upload request responses will be saved.
- `tmp_path`: This is only needed for selected text upload.
- `dump_path`: Path to where all of the response headers will be written to by CURL.

```lua
local M = {}

M.config = {
  db_path = vim.fn.stdpath("state") .. "/paste.db.json",
  tmp_path = "/tmp/paste",
  dump_path = "/tmp/dump", -- needs to match with the scripts dump path (explained later)
}
```

As, for the `setup` function we now need to extend the default configuration table with the
current one (the one i.e. passed in through the setup function). We do this because
if we ever decide to change the configuration on the fly, then this will be useful and
we won't need to reload Neovim again.

```lua
function M.setup(options)
  options = vim.F.if_nil(options, {})
  M.config = vim.tbl_deep_extend("keep", options, M.config)
  M._db = Path:new(M.config.db_path)
  M._dump = Path:new(M.config.dump_path)
  M._path = Path:new(M.config.tmp_path)
  M._responses = {}
  -- load db entries (persist)
  if M._db:exists() then M._responses = vim.json.decode(M._db:read()) end
end
```

After this, we initialize some private tables for several caching and file IO operations.

- `M._response`: This table is used for storing responses. For instance a response from https://0x0.st
  might look like the following.

  ```
  HTTP/2 200 
  server: nginx
  date: Mon, 11 Mar 2024 16:06:07 GMT
  content-type: text/html; charset=utf-8
  content-length: 24
  x-expires: 1741668317786
  strict-transport-security: max-age=63072000; includeSubDomains; preload
  x-frame-options: sameorigin
  x-content-type-options: nosniff
  x-xss-protection: 1; mode=block
  referrer-policy: no-referrer, strict-origin-when-cross-origin
  ...
  ```
  We would then parse this to a <samp>key: value</samp> pair value and then encode
  these into a JSON format. See: [<samp>M.pretty()</samp>]({{< ref "#hl-5-1" >}} "Response prettying function").
- `M._dump`: `Path` instance of `M.config.dump_path` where responses from CURL will
  be written to (by CURL itself) before parsing into a <samp>key: value</samp>.
  Again, we will be reading and writing to it a lot so, it's better to make it
  a pair value.
- `M._db`: `Path` instance of `M.config.db_path`.
- `M.tmp_path`: The path where currently selected buffer contents will be copied to.

### Upload

So for this we just need to follow the following steps.

- Prepare the file `M._path` for upload by writing all the buffer contents to it.
- Then upload the newly updated file to https://0x0.st using a job API.
- Save the returned link into the clipboard.
- Save the response dumped into `M._dump` to the `M._responses` table.
- Clean `M._path` and `M._dump`.
- Save all recorded `M._responses` upto this point to `M._db` path.

```lua
function M.paste(contents)
  M._path:write(table.concat(contents, "\n"), "w")
  Job:new({
    command = "curl",
    args = { -- read the manpage - I can't be fucked.
      "--silent",
     "--show-error",
     "--location",
     "--dump-header",
      M._dump.filename,
     "--compressed",
     "--request",
     "POST",
     "--form",
      string.format("file=@%s", M._path.filename),
     "https://0x0.st",
    },
    on_exit = vim.schedule_wrap(function(self, code, _)
      if code ~= 0 then return end
      local body = self:result()[1] -- [1] because it only returns a link
      local pretty = M.pretty(M._dump:read()) -- defined below
      M._responses[body] = {
        body = body,
        headers = pretty.headers,
        code = pretty.code
      }
      -- clean used paths (optional)
      M._path:rm()
      M._dump:rm()
      vim.fn.setreg("+", body)
      vim.api.nvim_notify("+pastebin: " .. body, vim.log.levels.INFO, {
        title = "pastebin",
        icon = " "
      })
      M._db:write(vim.json.encode(M._responses), "w")
    end),
  }):start()
end
```

CURL returns results in a [weird]({{< ref "#hl-3-1" >}} "See CURL response output") format,
so next we will be writing a `M.pretty` function that will convert these values into a
Lua table. Which then can be encoded to a JSON format.

> You could define `M.pretty` as a local function if you want.

```lua
function M.pretty(data)
  local lines = vim.split(data, "\r\n", { plain = true })
  local headers = {}
  local heading = vim.split(lines[1], " ", { plain = true })
  local code = heading[2]
  table.remove(lines, 1)
  for _, line in ipairs(lines) do
    local _line = vim.split(line, ": ", { plain = true })
    if #_line > 1 then
      local _colon_items = vim.split(_line[2], "; ", { plain = true })
      if #_colon_items == 1 then
        headers[_line[1]] = _colon_items[1]
      elseif #_colon_items > 1 then
        headers[_line[1]] = _colon_items
      end
    end
  end
  return { code = code, headers = headers }
end
```

### Delete

For deleting a link form https://0x0.st, we just need to send the link, that we want
to delete and its token that was given to us when we uploaded the file.

```lua
function M.delete(response)
  if not response then return end
  -- 0x0 will not allow deletion if we are unable to supply the token
  if not response.headers["x-token"] then
    M._responses[response.body] = nil -- rm from cache? This depends on your preference.
    M._db:write(vim.json.encode(M._responses), "w") -- useless if you chose to rm previous line
    return
  end
  Job:new({
    command = "curl",
    args = {
      "--form",
      string.format("token=%s", response.headers["x-token"]),
      "--form",
      "delete=",
      response.body, -- file link to delete example: https://0x0.st/HkmZ.lua
    },
    on_exit = vim.schedule_wrap(function(self, code, _)
      if code ~= 0 then return end
      -- I use https://github.com/rcarriga/nvim-notify
      -- if you don't then just use print("Deleted" .. response.body)
      vim.api.nvim_notify("Deleted " .. response.body, vim.log.levels.INFO, {
        title = "pastebin",
        icon = " "
      })
      M._responses[response.body] = nil
      M._db:write(vim.json.encode(M._responses), "w") -- save cache (persist)
    end),
  }):start()
end
```

## Debloat. Rewrite.

Why though?

- Will you only ever upload code files?
- Won't you want to upload a file without opening Neovim?
- Wouldn't it be better if there were a nice shell utility that can be used everywhere?
- Minimalism.

So, we write a shell script that does just that. (Maybe we'll write completions for it as well!)

```sh
#!/usr/bin/env bash

dump="/tmp/dump"

function help_message() {
  echo 'usage: paste [[--upload|-u]|[--delete|-d]|[--clean|-c]|[--help|-h]]'
  echo
  echo 'script to upload files to a pastebin website'
  echo
  echo 'options:'
  echo '  -h, --help                                    show this help message and exit'
  echo '  -c, --clean                                   remove db'
  echo '  -u <FILE>, --upload <FILE>                    upload a file to 0x0.st'
  echo '  -d <TOKEN> <LINK>, --delete <TOKEN> <LINK>    delete an uploaded file'
  echo 
  echo 'Source: https://github.com/dharmx'
}

function _upload() {
  [[ "$1" == "" ]] && echo Needs '<FILE>' && help_message && return 1
  curl                      \
    --silent                \
    --show-error            \
    --location              \
    --dump-header "$dump"   \
    --compressed            \
    --request POST          \
    --form file=@"$1"       \
      https://0x0.st
}

function _delete() {
  [[ "$1" == "" || "$2" == "" ]] && echo Needs '<TOKEN> <LINK>' && help_message && return 1
  curl --form token="$1" --form delete= "$2"
}

case "$1" in
  --upload|-u) _upload "$2" ;;
  --delete|-d) _delete "$2" "$3" ;;
  --clean|-c) rm -rf "$dump" ;;
  --help|-h) help_message ;;
  *) echo -e Needs arguments! "\n" && help_message && exit 1 ;;
esac
```

We then place this into `$PATH`. I recommend `~/.local/bin`. Now, you can use `0x0 -u file`.

```sh
#compdef 0x0

# Sing with me. WE :clap:. LOVE :clap:. COMPLETIONS :clap:!
function _0x0() {
  _arguments                                                    \
    '-h[show this help message and exit]'                       \
    '--help[show this help message and exit]'                   \
    '-c[remove db]'                                             \
    '--clean[remove db]'                                        \
    '-u[upload a file to 0x0.st]{?:}upload:_files'                 \
    '--upload[upload a file to 0x0.st]{?:}upload:_files'           \
    '-d[delete an uploaded file]'                               \
    '--delete[delete an uploaded file]'
}
```

We then place this into `$FPATH`. I recommend `~/.local/share/zsh/completions`.
Name the file `0x0` or, something :shrug:.

### Rewritten

Only the `M.paste` and `M.delete` functions need to be re-written with new arguments. See
below.

```lua
-- ...
function M.paste(contents)
  M._path:write(table.concat(contents, "\n"), "w")
  Job:new({
    command = "0x0", -- the script's name
    args = { "--upload", M._path.filename },
    on_exit = vim.schedule_wrap(function(self, code, _)
      if code ~= 0 then return end
      local body = self:result()[1]
      local pretty = M.pretty(M._dump:read())
      M._responses[body] = { body = body, headers = pretty.headers, code = pretty.code }
      M._path:rm()
      M._dump:rm()
      vim.fn.setreg("+", body)
      vim.api.nvim_notify(body, vim.log.levels.INFO, { title = "0x0", icon = " " })
      M._db:write(vim.json.encode(M._responses), "w")
    end),
  }):start()
end

function M.delete(response)
  if not response then return end
  if not response.headers["x-token"] then
    M._responses[response.body] = nil
    M._db:write(vim.json.encode(M._responses), "w")
    return
  end
  Job:new({
    command = "0x0",
    args = { "--delete", response.headers["x-token"], response.body },
    on_exit = vim.schedule_wrap(function(self, code, _)
      if code ~= 0 then return end
      vim.api.nvim_notify(vim.inspect(self:result()), vim.log.levels.INFO, { title = "0x0", icon = " " })
      M._responses[response.body] = nil
      M._db:write(vim.json.encode(M._responses), "w")
    end),
  }):start()
end
-- ...
```

## Command

There are a few rules we'll follow.

- `<bang>` should not allow an file path arguments.
- Without `<bang>` allow file path arguments.
- If no `<bang>` or, any arguments are supplied, then use current buffer.
  And, only allow ranges in this case.

```lua
-- at the very end of your paste.lua
function M.command(args)
  if args.bang and args.fargs[1] then
    M.delete(M._responses[args.fargs[1]])
  elseif args.fargs[1] then
    M.paste(vim.fn.readfile(file))
  else
    args.line1 = (args.range == 2 and args.line1 or 1) - 1
    args.line2 = args.range == 2 and args.line2 or -1
    M.paste(vim.api.nvim_buf_get_lines(0, args.line1, args.line2, false))
  end
end
```

Now, you'd just import the `paste` module and call the `command` function and
pass in the `args` parameter.

See: [<samp>:help nvim_create_user_command</samp>](https://neovim.io/doc/user/api.html#nvim_buf_create_user_command())

> I have it in `~/.config/nvim/lua/scratch/paste.lua` so, I'd need to call `require("scratch.paste")`.
  If you have it in say, `~/.config/nvim/lua/paste.lua` then you'd just need to do `require("paste")`.

```lua
-- in your init.lua
vim.api.nvim_create_user_command("Paste", function(args)
  require("scratch.paste").command(args)
end, {
  desc = "Upload/delete files/snippets to a pastebin site.",
  range = true,
  bang = true,
  nargs = "?",
  complete = function(arg, name, _)
    if name:match("^'<,'>Paste") then return {} end
    if name:match("^Paste!") then
      return vim.tbl_keys(require("scratch.paste")._responses)
    end
    return vim.fn.getcompletion(arg, "file")
  end,
})
```

The custom command completion function is not complicated at all. We just check

- If the dumbass is using ranges. If yes then do not supply any completions.
- If the _restarted_ individual is using `<bang>` i.e. `:Paste!`. If they are, then supply
  all response bodies list, which the user would want to delete.
- Else supply with file paths.

## Conclusion

{{< admonition warning "Please read TOS" true >}}<!--{{{-->
Also, please do not forget to read the [TOS](https://0x0.st/). Like the maximum file
size you're allowed to upload. Try not to upload furry
porn to it. You can use [mega.io](https://mega.io/) for that. And, please no gore or,
propaganda videos.

Just go and [read](https://0x0.st/) it before you do anything stupid.
{{< /admonition >}}<!--}}}-->

I hope you enjoyed reading this meme :skull:. If you did then comment "Sex" for 25 Robux.
