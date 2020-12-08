What's in a prompt?
####################################

:date: 2019-12-07 14:40
:category: fish

There's something weird about shell prompts. You'll see them described as "minimalist" or featureful, and you'll see them as multiline, single-line, monochrome or lit up like a christmas tree, but they'll all incorporate the same 10 or so bits of information:

- The working directory - where the shell currently "is"
- The prompt symbol - a `$` or a `#` or a `>`, in fancier prompts a unicode symbol
- The last status - what the last program returned
- The current VCS status - what's the git branch?
- Any other "global tool state" - virtualenv, ruby env
- Vi-mode - if you use vi-style bindings, this is quite important [#]_
- the time

My own prompt also features a battery indicator and used to show the currently playing song in mpd, but those are unusual.

.. [#] So important in fact that fish automatically enables it for you.
