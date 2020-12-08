The best FAQ doesn't exist
####################################

:date: 2019-07-16 20:20
:category: fish

In fish < 3.0, we've had a frequently asked question [#]_ that adding bindings in your config.fish (the configuration file) didn't work.

You see, fish doesn't use the common "readline" library, but has its own input/output facilities, and to bind keys you'd just execute a `bind` command like

.. code-block:: fish

    bind \cg forward-word
    bind \t complete

And that works if you're doing it interactively, so you like the binding and add it to your config file, and restart fish and... the bindings are gone?

This was a pretty frequent question for us, and we'd always answer the same:

  You can't add bindings in config.fish, you'll have to add them to a function called `fish_user_key_bindings`.

The reason for this was a pretty fundamental decision that:

- We'd support an emacs-inspired "default" mode and a vi-inspired "vi" mode [#]_
- Switching between the two would leave you with the same bindings every time
- Setting up the mode could be done by setting a variable, global or universal [#]_

Because of all these, what we did was to load the bindings *after* reading the config file (because the variable could be set there), and because of the second, we'd *erase* all bindings whenever a binding mode was picked. The result was changes from the config file disappearing.

Now, fixing this took me quite a while, and the fix turns out to be surprisingly elegant and simple:

- There are now "preset" and "user" bindings
- Modes set up their bindings as preset bindings
- Switching mode just erases the preset bindings

That means if you tried out vi-mode and decide to go back, it just erases all the h/j/k/l and such bindings and leaves your custom stuff untouched.

This has the further advantage that you can now decide to remove a custom binding and it'll leave the preset.

But what's best about this is that I no longer have to answer the FAQ. It just simply stopped existing.

.. [#] That should have been an actual FAQ entry. Sorry!
.. [#] And theoretically other modes, someone just has to write them.
.. [#] Universal variables are persistently stored variables - fish saves them to disk, and shares them among the running sessions.
