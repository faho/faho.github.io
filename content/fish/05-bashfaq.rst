BashFAQ through fish's eyes
####################################

:date: 2019-12-07 14:40
:category: fish

So here's one thing I wanted to do for a while. Let's look at the `BashFAQ <http://mywiki.wooledge.org/BashFAQ>`_ and see how it holds up in fish.

This is not a comprehensive comparison (in particular the BashFAQ is by its nature something of a list of bash's most obvious problems) and I'm going to skip over uninteresting questions (including those that are substantially similar in both and those that deal with certain commands). Since much of this is down to Bash following POSIX most of the bash answers apply to e.g. dash and zsh as well.

This isn't meant to bash (hehehe) bash, but more of an explanation of why we're bothering with our own scripting language at all. One of the most common comments whenever fish is discussed is "well, it's not POSIX compatible", so this is, in part, an attempt to show *why*.

1. How can I read a file (data stream, variable) line-by-line (and/or field-by-field)?

   The bash answer: Don't try to use "for". Use a while loop and the read command (specifically `read -r` with IFS unset)

   The fish answer: A for-loop is fine (`for line in (cat file)`), as is `while read line`.

   Fish doesn't do word-splitting the way bash does - it splits command substitutions on newlines, and while `read` obeys $IFS it is the only part of fish to do so [#]_ so $IFS isn't typically changed. If you want to be super-pedantic set $IFS to `\n\ \t` before doing anything.

4. How can I check whether a directory is empty or not? How do I check for any .mpg files, or count how many there are?

   The bash answer: Set nullglob, dotglob and an array. Possibly use subshells.

   The fish answer: Use `count * .*`. Fish uses nullglob behavior for that, and doesn't have dotglob.

   The nice part here is that the answer is *always* valid, and that you don't have to remember to set an option, possibly, if you haven't already set it.
   
5. How can I use array variables?

   The fish answer: Use `set` to set/erase an "array" [#]_ or any of its elements, use `$var` to expand it fully and `$var[slice]` to expand some of its elements (including `$var[1..3 7..-2]`). Overall fish defaults to lists as first-class things, while bash added them onto POSIXoid syntax, so it needs to do a bunch of contortions.

7. Is there a function to return the length of a string?

   The bash answer: `${#varname}`

   The fish answer: `string length -q -- $varname`.

   This is simply a case where fish prefers to use commands, while bash prefers syntax.

20. How can I find and safely handle file names containing newlines, spaces or both?

    The bash answer: Complicated. Quote, use `--`, use globs or `find -print0`.

    The fish anwer: Fish doesn't do word-splitting on variables and only splits command substitutions on newline, so you don't have to worry about spaces at all (apart from quoting them in literal arguments). Bash's `read` also features a `-d` option that can be passed a NULL as a delimiter via `$'\0'`. In fish you'd use `read -z`. Also since fish doesn't put while-loops into subshells you can set variables inside them. Also since fish 3.0 `string split0` is available to split an entire command substitution on NULLs.

    .. code-block:: fish

       for file in (find . -print0 | string split0)

    works.

22. How can I calculate with floating point numbers instead of just integers?

    The bash answer: Bash's arithmetic does ints only, use `bc`.

    The fish answer, since fish 3.0: Use `math`, and `test` even for floating comparisons. Fish's `math` used to be a function that called `bc` in the background, but `bc` has some major issues. For one macOS (being the paragon of current software that it is) ships a slightly-outdated version of bc that has problems with large results [#]_ and bc can't do modulo and floats at the same time [#]_. Also `bc` does a heck of a lot of stuff and we can't document all of it so there's always this weird corner where we'd have to send people to external documentation. So we wrote our own `math` [#]_, and it should *just work*.

24. I set variables in a loop that's in a pipeline. Why do they disappear after the loop terminates? Or, why can't I pipe data to read?

    The bash answer: Subshells.

    The fish answer: They don't, unless you've scoped them to be local to the loop. Any variable defined with `set -l` inside will disappear, but put that outside and `set` it inside and it stays. No subshells here.

25. How can I access positional parameters after $9?

    The bash answer: Use ${10} instead of $10. For other shells you might have to use `shift`.

    The fish answer: Use `$argv[10]`.

35. How can I handle command-line options and arguments in my script easily?

    The bash answer: Either manual parsing ("the most flexible approach, and is sufficient for most scripts. It is the best way, really") or `getopts`, never `getopt` (because that doesn't do long options). The manual approach makes it hard to use grouped short options (like in `ls -lah`).

    The fish answer: `argparse`. This is one tool that was written explicitly as a reaction to questions about how to parse arguments.

    It's probably easiest to demonstrate with an example. So let's see how fish's `abbr` function handles its arguments:

    .. code-block:: fish

     set -l options --stop-nonopt --exclusive 'a,r,e,l,s,q' --exclusive 'g,U'
     set -a options h/help a/add r/rename e/erase l/list s/show q/query
     set -a options g/global U/universal

     argparse -n abbr $options -- $argv
     or return

     if set -q _flag_help
         __fish_print_help abbr
         return 0
     end

    As you see, `argparse` gets a list of options to handle (with long and short versions), and then the arguments to parse. It puts the found options into $_flag_XYZ variables and leaves the positional arguments in $argv.
    
    This takes care of all the option ungrouping, finding option parameters and such, and can even be used to disallow option combinations and require a minimum or maximum number of arguments.
    
37. How can I print text in various colors?

    The bash answer: Use `tput`, like `blue=$(tput setaf 4)`.

    The fish answer: Use `set_color`. Of course `tput` would also work, but it's a bit weird to require an external tool for this [#]_, `set_color` can take RGB or named colors [#]_.
    
39. What are all the dot-files that bash reads? 

    The bash answer: Depends on whether your shell is interactive and/or a login shell. Sometimes there are multiple options ("/etc/profile and then one of .bash_profile or .bash_login or .profile").

    The fish answer: /etc/fish/config.fish and ~/.config/fish/config.fish [#]_. Always. In an interactive shell, a login shell, on tuesdays, when it's raining outside and whether your system is Debian or Solaris.

    This is one point where I'd argue it's a bit of an over-simplification. It's obvious to me that login-shells aren't special or useful enough to deserve their own config file (in fish you can guard bits behind `if status is-login` and that seems entirely enough to me), but I think reading config.fish in non-interactive shells causes issues if you're using it as a script interpreter (via `#!/usr/bin/env fish`).

    So personally I'd prefer if we didn't read config.fish in that case, but I don't have an answer as to what we should then read. Bash would do well with some simplification however.
    
41. How do I determine whether a variable contains a substring?

    The bash answer: Use `if [[ $foo = *bar* ]]`

    The fish answer: You want to do something with strings? Use `string`. Here `string match -- '*bar*' $foo` would do.
    
46. I want to check to see whether a word is in a list (or an element is a member of a set).

    The bash answer: Use associative arrays (bash >= 4) or for-loops.

    In this case the associative array seems to be used purely for performance reasons and doesn't actually simplify the code (because you still need to loop through the things to set the array).

    The fish answer: Use `contains`.
    
    .. code-block:: fish

      read input
      if contains -- $input Bigfoot UFOs Republicans
          echo $input exists
      else
          echo $input does not exist
      end
    
51. I want history-search just like in tcsh. How can I bind it to the up and down keys?

    The bash answer: Just add the following to /etc/inputrc or your ~/.inputrc:

    .. code-block:: sh

      "\e[A":history-search-backward
      "\e[B":history-search-forward

    The fish answer: It's bound like that by default.

    This is one of the first things I add whenever I use bash anywhere. It baffles me why it's not the default.
    
66. I want to check if [[ $var == foo || $var == bar || $var == more ]] without repeating $var n times.

    The bash answer: Use `case` or extglobs.

    The fish answer: Use `contains` or `case`.

67. How can I trim leading/trailing white space from one of my variables?

    The bash answer: Use extglobs or incantations like

    .. code-block:: sh

      junk=${var%%[! ]*}   # remove all but leading spaces
      var=${var#"$junk"}   # remove leading spaces from original string
      
      junk=${var##*[! ]}   # remove all but trailing spaces
      var=${var%"$junk"}   # remove trailing spaces from original string

    The fish answer: Use `string trim`.

73. How can I use parameter expansion? How can I get substrings? How can I get a file without its extension, or get just a file's extension? What are some good ways to do basename and dirname?

    The bash answer: Use parameter expansion.

    The fish answer: Use `string`. This also applies to FAQ 100, which explicitly asks for string manipulation.

118. How do I print the contents of an array in reverse order, or reverse an array?

     The bash answer:

     .. code-block:: sh

        idx=("${!a[@]}")
        b=()
        for (( i=${#idx[@]} - 1; i >= 0; i-- )); do
          j=${idx[i]}
          b+=("${a[j]}")
        done


     The fish answer: `set b $a[-1..1]`.

So, what have we learned here? Fish builds most of its scripting power on its builtins, not its syntax, and it has builtins made to solve actual problems that people have. `math` does computations, `string` does string-handling, `argparse` parses args. Also "arrays" are nicer to work with because they're first-class instead of an afterthought [#]_.

Overall I think we're doing okay.

.. [#] And we want to remove it.
.. [#] We standardized on calling it a "list" a while ago because that's a more normal english word, but it's the same idea.
.. [#] It's been a while, but I think it insists on splitting them into multiple lines with a backslash at the end of each? Which makes it not a valid number anymore for other tools, which means it's quite dangerous to use.
.. [#] Seriously. `echo "5 % 2" | bc -l` prints "0".
.. [#] Based on the tinyexpr library, which we modified extensively.
.. [#] That may not be installed - NetBSD doesn't have it.
.. [#] And handles how many colors the terminal supports.
.. [#] Okay, technically these are SYSCONFDIR/fish/config.fish and $XDG_CONFIG_HOME/fish/config.fish.
.. [#] Because coming from POSIX sh, arrays *are* an afterthought.
