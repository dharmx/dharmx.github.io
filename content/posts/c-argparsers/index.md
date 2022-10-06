---
title: "C Argument Parsers"
subtitle: "Parsing command line arguments in C."
date: 2022-10-12T21:06:05+05:30
lastmod: 2022-10-12T21:06:05+05:30
draft: true
author: "dharmx"
authorLink: "https://dharmx.is-a.dev"
summary: "Finding feature rich argument parsers is hard in C. This is because there aren't many.
          This article describes my experience with some of them like GNU/getopt and the C argparse repository."
description: "Building some command line programs that will perhaps demostrate how some common argument parsers work in C."
license: "<a rel='license external nofollow noopener noreffer' href='https://opensource.org/licenses/GPL-3.0' target='_blank'>GPL-3.0</a>"
images: ["/c-argparsers/images/featured-image.png"]

tags: ["c", "libs", "argparsers"]
categories: ["coding"]

featuredImage: "images/featured-image.png"
featuredImagePreview: "images/featured-image.png"

hiddenFromHomePage: false
hiddenFromSearch: false
twemoji: true
lightgallery: true
ruby: true
fraction: true
fontawesome: true
linkToMarkdown: true
rssFullText: false

toc:
  enable: true
  auto: true
---

# Motive

Hey, there. For some reason, I had a sudden urge for learning C seriously. In the
past, I have tried numerous times, and you could say that I found reasonable
success in doing so. But, the knowledge just wouldn't stick. I would end up
forgetting most of the stuff pretty quickly. And this does not apply just to C,
it applies to every difficult topic that I tried to learn. But, after a few
self-experimentation, I have found a remedy. Although unsure but it lately
seemed to work.

This is a record of one of such experiments where I challenge myself to explore
argument parsers for C. I'm well acquainted with them in Python so, wanted to
know how feature-rich they are in C.

## Definition

Argument parsers are a convenient way of handling command-line arguments for your
program. Just like libraries, argument parsers provide utilities for managing
arguments for your CLI application. Command line parsers differ in the number of
features they provide. Some have a lot of features while some are extremely
minimal.

For instance, such features may include the following.

- **Short options**: `-c`, `-A` i.e. they generally contain one letter.
- **Long options**: `--version`, `--enable` i.e. they generally have a word or, 
  multiple words joined with symbols like `--load-plugins`, `--fetch_new_database`,
  etc.
- **Chain options**: `-Syyu` translates to `--sync --refresh --quiet --sysupgrade`
  i.e. they are executed in succession.
- **Argument types**: These will act as a hint to the developer as to how to convert
  that argument. For example, `--notify` takes in either `true` or, `false`.
  So, it makes sense that the program tries to check and convert to a `boolean`
  datatype in that language. This way you be able to handler any outliers, and
  it would be easier to generate the help message as well.
- **Sub-options**: Sub-options are options require an option to be specified
  before it can be used. It is mainly used to enable or, disable a functionality
  that is granted by the main option. Additionally, it can also be used to
  categorize options like for example a CLI music player might have an option
  related to `playback` which would have sub-options like `seek`, `pause`,
  `play`, etc. And, `audio` would have `increase-volume`, `mute`, etc. These
  would be issued in the following way.

  ```sh
  player --audio --mute
  player --playback --rewind --seek 12:20
  ```

- **Option descriptions**: Some argument parsers also allow specifying description
  of what an option does which help generating the help message and sub-command
  help messages. An example of such parse is Python's built-in [argparse](https://docs.python.org/3/library/argparse.html)
  library.

These are just few examples of what argument parsers might provide APIs for.

## Utilities

Moving on, let's not forget that this is a C specific meaning, that we will be
looking at argument parser libraries and functions that are available in C only.
We will be writing few minimal programs to illustrate the usage each of those
libraries.

Now, we shall be listing down the libraries and functions we will be using.

- The `getsubopt()` function.
- The `getopt_long()` function.
- The `getopt_long_only()` function.
- The [cofyc/argparse](https://github.com/cofyc/argparse) library.
- The [GNU/argp](https://www.gnu.org/software/libc/manual/html_node/Argp.html)
  and [GNU/getopt](https://www.gnu.org/software/libc/manual/html_node/Getopt.html)
  utilities.

I wanted to explore more libraries like [libucw](http://www.ucw.cz/libucw) but,
the article would be quite lengthy. Well, maybe I might when I write another
article about this topic (well, not this, but it would be about making my own
argument parser library).

## GNU/getopt

Well, I should start this off by saying that I will be using Linux for this
(and you should too). Anyway, `getopt` comes with Linux i.e. the `unistd.h`
header to be exact. It uses global variables to keep track of the current
state of the parser. Such variables include `optopt`, `opterr`, `optind`,
and `optarg`. These will be explained shortly.

```c
#if defined(__GNU_LIBRARY__) || defined (WIN32) || defined (__CYGWIN__)
/* Many other libraries have conflicting prototypes for getopt, with
   differences in the consts, in stdlib.h.  To avoid compilation
   errors, only prototype getopt for the GNU C library.  And not when
   compiling with C++; g++ 4.7.0 chokes on conflicting exception
   specifications.  */
#if !defined (__cplusplus)
extern KPSEDLL int getopt (int argc, char *const *argv, const char *shortopts);
#endif
#if defined (__MINGW32__) || defined (__CYGWIN__)
#define __GETOPT_H__ /* Avoid that <unistd.h> redeclares the getopt API.  */
#endif
#elif !defined (__cplusplus)
extern KPSEDLL int getopt ();
#endif
```

Right from the header we can see that `getopt` takes in an integer value
which happens to be the length of the number of arguments or, rather
the length of the array `argv`. Secondly, it will take in the actual
arguments. Third, it will take an array of configurations which will
specify which argument characters are allowed and if they take in
any arguments or, not.

The configuration options are as follows.

- `"abc"`: This means that the arguments `-a`, `-b` and `-c` will not
         allow any arguments at all. Meaning, `./app -a 23` would
         result in an immediate error.
- `"a:b"`: A colon means that the argument letter before
         that requires an argument. Failing to do so will result
         in an error. For example, `./app -a` would result in an
         error. And, `./app -a 'hello'` will work.
- `"a::b:c"`: This means that `-c` does not require an argument, `-b`
            requires an argument. Lastly, `-a` will work without
            an argument. Meaning, `./app -a hello`, `./app -a` will
            both work.

The arguments are passed by `getopt(argc, argv, "a::b:c")`, and it will
return the option character itself if it is in the configuration string.
If it reaches the end then it will return `-1`.

### Global Variables

The `getopt` function has an internal state, meaning it uses some global
variables to keep track of the current state of arguments.

#### The `optarg` global

The `optarg` global variable is updated when an option with
argument is encountered. We can then parse that argument and use it in our
program.

```c
/* ./app -a -b Hank -c */
/* ./app -a 78 */
int main(int argc, char** argv) {
    int option;
    while ((option = getopt(argc, argv, "a::b:c")) != -1) {
        switch (option) {
            case 'a': printf("Hello, %s!\n", optarg != NULL ? optarg : "John"); break;
            case 'b': printf("Hey, %s.\n", optarg); break;
            case 'c': printf("Greetings!\n"); break;
            default: printf("Default!\n"); break;
        }
    }
}
```

From the above snippet you can see that we are using `optarg` for getting
the arguments. `getopt` parses `argv` and puts the arguments into `optarg`
if it satisfies the constraints imposed by the configuration string i.e.
`"a::b:c"` in this case.

#### The `optopt` global

The `optopt` global variable will be updated whenever an unknown option
is passed. Hence, consider the following snippet.

```c
while ((option = getopt(argc, argv, "a::b:cdz")) != -1)
    printf("%c\n", optopt);
/*
./app: invalid option -- 'g'
g
*/
```

Simple. When you attempt to pass `-g` for example, the program will
print out an error message like the following.

If for some reason one decides to make use of the unknown option
then they can use `optopt`. The reason we are not using the `option`
local variable is because `?` will be returned by `getopt` whenever
an unknown option is being passed. Although, we can make use of
the `?` by adding it to the `if/switch` statement.

```c
while ((option = getopt(argc, argv, "ab")) != -1) {
    switch (option) {
        case 'a': printf("First!\n"); break;
        case 'b': printf("Second!\n"); break;
        case '?': printf("Unknown: %c!\n", optopt); break;
        default: printf("Undefined!\n"); break;
    }
}
```

#### The `opterr` global

If the value of this variable is nonzero, then `getopt` prints an error
message to the standard error stream if it encounters an unknown option
character or an option with a missing required argument. This is the
default behavior. If you set this variable to zero, `getopt` does not
print any messages, but it still returns the character `?` to indicate an
error.

This can be useful if you do not like the default error message that
gets printed by `getopt` and want to write your own custom error
message, like the following.

```c
int option;
opterr = 0;
while ((option = getopt(argc, argv, "-a::b:c")) != -1) {
    if (option == '?')
        printf("ERROR: -%c is an unknown option!\n", optopt);
}
/*
./app -lol
ERROR: -l is an unknown option!
ERROR: -o is an unknown option!
ERROR: -l is an unknown option!
*/
```

#### The `optind` global

This variable is set by `getopt` to the index of the next element of the
`argv` array to be processed. Once `getopt` has found all the option
arguments, you can use this variable to determine where the remaining
non-option arguments begin. The initial value of this variable is `1`.
This can be used to manually get the argument of an option.

```c
int option;
opterr = 0;
while ((option = getopt(argc, argv, "a::b:c")) != -1)
    printf("INFO: %s\n", optind < argc ? argv[optind] : "(null)");

printf("INFO: %s\n", optind < argc ? "rest of the arguments..." : "end");
int index = optind;
while (index < argc) {
    printf("INFO: %d place: %s\n", index, argv[index]);
    ++index;
}
/*
./a.out -lol this is the rest
INFO: -lol
INFO: -lol
INFO: this
INFO: rest of the arguments...
INFO: 2 place: this
INFO: 3 place: is
INFO: 4 place: the
INFO: 5 place: rest
./a.out -l
INFO: (null)
INFO: end
*/
```

### Word Counter

We will now be creating a minimal clone of the GNU coreutil `wc` using
what we have learned i.e. by using `getopt`. This clone will only
have character count, line count and print help message features.
Hence, parse the following code.

```c
/* cc wo.c -o wo */
#include <stdbool.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int
count(char match, bool all) {
    int count = 0;
    while (true) {
        char symbol = getchar();
        if (symbol == EOF) return count;
        if (all || symbol == match) ++count;
    }
    return count;
}

int
main(int argc, char** argv) {
    // optional but clean
    enum { CHR_CNT = 'c', LN_CNT = 'l', ERR_SIGN = '?', HELP_MSG = 'h' };

    int option;
    opterr = false;
    while ((option = getopt(argc, argv, "clh")) != -1) {
        switch (option) {
            case CHR_CNT: printf("%d\n", count('*', 1)); break;
            case LN_CNT: printf("%d\n", count('\n', 0)); break;
            case HELP_MSG: printf("wo -[c|l]\n"); break;
            case ERR_SIGN: perror("unknown option"); break;
            default: perror("undefined error"); break;
        }
    }
    return EXIT_SUCCESS;
}

// vim:filetype=c
```

You may run this by using `cat` and piping the `STDOUT` into the `wo`.
Additionally, you can also use `<<<` for a string and `<` for a file.
Learn more about shell redirections from [here](https://www.redhat.com/sysadmin/redirect-shell-command-script-output)
and [here](https://stackoverflow.com/questions/18515135/triple-less-than-sign-in-bash).

```sh
# test using the source file
cat wo.c | ./wo -c
888
./wo < ./wo.c
888
# test using a string
echo 'hello world!' | ./wo -l
1
./wo -l <<< 'hello there!'
1
```

Chaining is another neat effect of `getopt`. Meaning, that you can pass
options and arguments without separating them with spaces. Like the
following example.

```sh
./wo -hhhc < wo.c
wo -[c|l]
wo -[c|l]
wo -[c|l]
888
```

{{< admonition note "Where can I get more test samples?" >}}
There are heaps of samples online. Well, to be honest you can
copy any text format file really. but here are some of the sites
where you can get sample text files.

1. [filesamples](https://filesamples.com/formats/txt)
2. [learningcontainer](https://www.learningcontainer.com/sample-text-file/)
3. [sample-videos](https://www.sample-videos.com/download-sample-text-file.php)
4. [textfiles](http://www.textfiles.com/)
{{< /admonition >}}

Okay! Moving on next I will be explaining `getopt_long` which I am really fond
of using and use it most of the time.

## GNU/getopt_long

This is an extended version of `getopt` that allows usage of both long and short
options. Now, let's look at the function prototype.

```c
#define no_argument		0
#define required_argument	1
#define optional_argument	2

extern int getopt_long (int ___argc, char *__getopt_argv_const *___argv,
			const char *__shortopts,
		        const struct option *__longopts, int *__longind)
       __THROW __nonnull ((2, 3));
```

From the above code we can see that `getopt_long` takes in `__shortopts` which is
a string that will contain short options and secondly the `__longopts` are
identified by something different i.e. rather that using just an array of
strings we will use an array of `struct option` structure. See the following
snippet.

```c
struct option
{
  const char *name;
  /* has_arg can't be an enum because some compilers complain about
     type mismatches in all the code that assumes it is an int.  */
  int has_arg;
  int *flag;
  int val;
};
```

This should be intuitive i.e. `name` will take in the long option string
`has_arg` will be any option out of `no_argument: 0`, `required_argument: 1`
and `optional_argument: 2`. The `flag` however can be used to signal that
the program is currently exploring a sub-option (for example). The `value`
field will be the short option alternative. See the following usage
example to get a general idea.

```c
int branch = 0;
struct option example;
example.name = "fetch";
example.has_arg = optional_argument;
example.flag = &branch;
example.val = 'g';
/* app --fetch=link */
```

It should be noted that `getopt_long` also uses `optarg`, `optopt`, etc.
After this point onwards, everything is more or less the same.

## GNU/getopt_long_only

## GNU/getsubopt

## GNU/argp

## cofyc/argparse

## References

## Ending Note
