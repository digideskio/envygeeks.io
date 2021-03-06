---
title: On Bash complete in Ruby.
tags:
  - ruby
  - bash-completion
  - bash
---

Your auto-completion is kind-of complicated.  You build it entirely in `bash` with the assumption that it provides more perf (and it probably does... it probably isn't to-be honest... based on what I can gander just from looking at it.) However, it's not very maintainable even if it does have more perf.  Why not use Ruby to generate the auto complete list for Ruby?

### The argument storage

So lets take this from the start, we need to be able to let users do something like `hello w` and get `this is my list`, we don't need to be fancy and deal only returning relevant lists ourselves, lets leave that to `compgen` we just need to store a list of values and ship that list. So lets start with a hash:

```ruby
list = {
  "_reply" => [
    "help",
    "hello",
  ],

  "help" => {
    "_reply" => [
      "hello",
    ]
  },

  "hello" => {
    "_reply" => [
      "help",
      "--my-world",
      "--no-my-world",
      "world",
    ],

    "help" => {
      "_reply" => [
        "world"
      ]
    }
  }
}
```

So every hash has a `_reply` key with an array and every hash can have (relatively) infinite other keys with hashes that each have their own `_reply`, so we can dig deeply into sub-commands in a command system.  We have no root key so that we can remain agnostic with our script and even alias our parent script and still get completion.  We will then serialize that into `bin/comp-list.pak` so that we can read and load it quickly, [`msgpack`](//msgpack.org/){:target="_blank"}{:rel="noopener noreferrer"} is far faster than JSON or YAML.

### Generate the list with Ruby

Now we need to pump out the lists to Bash so that it can do what it needs to do for us.  Again, we do nothing fancy as we leave the sorting out of stuff to `compgen`. We can do that with this:

```ruby
#!/usr/bin/env ruby
# Frozen-string-literal: true
# Copyright: 2016 Jordon Bedwell - MIT License
# Encoding: utf-8

ARGV.shift
require "msgpack"
list = MessagePack.unpack(File.read(File.expand_path(
  "comp-list.pak", __dir__
)))

# --
# @param obj [Hash] the hash you wish to pull the `_reply` from.
# @param key [Any_] the string you are checking inside of the list.
# Check if a key is included inside of given reply object.
# @return TrueClass|FalseClass true|false
# --
def key?(obj, key)
  obj["_reply"].include?(
    key
  )
end

# --
# Checks to see if the key is partially within an array.
# @param obj [Hash] the hash you wish to pull the `_reply` from.
# @param key [Any_] the string grepping out of the list.
# @return TrueClass|FalseClass true|false
# --
def contains?(obj, key)
  result = obj["_reply"].grep(
    /#{Regexp.escape(key)}/
  )

  !result.empty?
end

# --
# Check if a key is an opt (argument).
# @return TrueClass|FalseClass true|false
# @param key [Any_] the key.
# --
def opt?(key)
  key =~ /\A-{1,2}/
end

# --

if ARGV.empty?
  $stdout.puts list["_reply"].join(
    " "
  )
else
  none = false
  rtrn = list

  ARGV.each_with_index do |key, index|
    if rtrn.key?(key) then rtrn = rtrn[key]
      elsif key?(rtrn, key) && !opt?(key) then none = true
      elsif index + 1 == ARGV.size && contains?(rtrn, key) then next
      elsif key?(rtrn, key) && opt?(key) then next
      else none = true
    end
  end

  unless none
    rtrn = rtrn["_reply"]
    $stdout.puts rtrn.join(
      " "
    )
  end
end
```

So, in short here is what we do:

- `bin\t\t` => `hello help`
- `bin h\t\t` => `hello help`
- `bin hello unknown\t\t` => `\n`
- `bin hello\t\t` => `help --my-world --no-my-world world`
- `bin hello --\t\t` => `--my-world --no-my-world`

And in short, here is what the script does before `compgen`:

- `bin\t\t` => `help hello`
- `bin h\t\t` => `help hello`
- `bin hello unknown\t\t` => `\n`
- `bin hello\t\t` => `help --my-world --no-my-world world`
- `bin hello --\t\t` => `help --my-world --no-my-world world`

### Tying it into Bash

Now it's time to tie it into Bash and `compgen`, so taking the list and the Ruby above and putting into all into a single file called `bin/comp-list` we can then turn around and create `bin/comp` with the following:

```bash
_bin() {
  comp=$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)/comp-list
  COMPREPLY=($(compgen -W "$($comp ${COMP_WORDS[@]})" -- \
    ${COMP_WORDS[COMP_CWORD]}))
}

complete -F _bin bin
```

### Bonus Round: Auto-auto-list for Thor

```ruby
# --
# Get the base.
# --
def base(const, skip = %w(help))
  keys = const.all_commands.keys
  return "_reply" => keys, "help" => {
    "_reply" => keys - skip
  }
end

# --
# Add the command options.
# --
def add_opts(out, const)
  const.all_commands.each do |key, val|
    val.options.map do |_, opt|
      out[key] ||= { "_reply" => [] }
      ary = out[key][
        "_reply"
      ]

      if opt.boolean?
        ary.push(opt.switch_name, "--no-#{opt.switch_name.gsub(
          /\A--/, ""
        )}")

      else
        ary.push(
          "#{opt.switch_name}="
        )
      end

      ary.push(
        *opt.aliases
      )
    end
  end

  out
end

# --
# Get a list of commands.
# --
def get_commands(const)
  out = base(const)
  if const.const_defined?(:Sub)
    then const.subcommands.each_with_object(out) do |key, hsh|
      hsh[key] = send(__method__, const::Sub.const_get(
        key.capitalize
      ))
    end
  end

  add_opts(
    out, const
  )
end

# --

namespace :gen do
  task :comp do
    require "msgpack"
    $stdout.puts(get_commands(My::CLI) \
      .to_msgpack)
  end
end
```
