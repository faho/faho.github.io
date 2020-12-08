Terminals are kinda bad
#######################

:date: 2020-12-08 19:00
:category: software


I spend a lot of my time in terminals.

That's probably a bit of an understatement. Really, basically all of my time on a computer is spent either in a browser, in a videogame or in a terminal. I like text-based interfaces, and I still don't know why I would use the emacs GUI.

So it probably won't be a surprise that I think our terminals are kinda... bad. Here's a few reasons why:

Arbitrary limitations
---------------------

Ever wondered why you can't bind Control+I separately from tab?

Well, the reason for that is ASCII. "I" is 73 (0x49), the tab character is 9 (0x09), and applying a control modifier masks off the upper 3 bits, so 73 - 64 = 9.
That means, as far as the application is concerned, there is no way to tell tab from Control+I. It is the same thing.

Similarly, there is no such thing as an "alt" key. There might be on your keyboard, but as far as your terminal is concerned it's really a fancy way of saying "escape". Pressing alt+e will send the escape character quickly followed by the "e" character. The way applications handle this is to wait for a bit after seeing an escape character to determine if it was an escape key that was pressed or an escape sequence.

This is one of the many many arbitrary limitations that our current terminals have that just aren't intrinsic to the text-based paradigm. There is no reason this needs to be that way, except that somebody in the 70s made it so and nobody has managed to change it.

Of course you could make a terminal that improves on this, but:

Terminal inconsistencies
------------------------

There are a lot of terminals, and a lot of variety in what they support and how they behave.

Some common differences:

- How many colors does it support?
- What escape sequences does it send for which keys?
- What optional features does it support? Can I send my PWD to it? Does it show a title?
- Does it reflow text on its own? VTE and iTerm do, konsole doesn't

To me, many of these differences just seem entirely pointless. Why does this send ``\x7f`` for my backspace key while that sends ``\b`` (the actual backspace character)? There is an answer, but it's a historical one rather than a technical one.

Another example: Konsole and iTerm both support cursor shaping (which is useful e.g. for vi-mode - make your cursor a block in normal mode and a ``|`` bar in insert mode). The sequences, as far as I can tell, work exactly the same [#]_, like cursor blinking), but one uses ``\e]50;CursorShape=0\x7``, while the other uses ``\e]1337;CursorShape=0\x7``. See the difference? Yeah, that's a "1337" in place of a "50". Cute. [#]_

But hey, you can surely detect these small differences automatically, right?

.. [#] (unlike the xterm one that provides other features
.. [#] Note: None of this is a slight against the Konsole or iTerm developers.
       I've had contact with at least iTerm's maintainer, and he's never been anything but lovely.

Detection sucks
---------------

Hahaha, no.

The primary way to detect what terminal your user is using and what it can do is $TERM, which is an environment variable that explains the type of terminal. You're supposed to use it to look at the actual terminal description in a thing called "terminfo". [#]_

Only terminfo is basically broken.

Why? Well:

- You need the actual files installed to be able to look it up, which means your OS needs to provide them
- People connect to servers via ssh, which just sends the $TERM value along, so all software running on that server will have to look up the terminfo for the terminal running on the (typically much newer) client, and if they don't find it they will fail to see anything and might even crash.
- Adding a new terminfo entry is possible, but with LTS distributions there's a *looong* lag. [#]_
- Some (arguably badly written) software just hardcodes a few $TERM values it knows, mostly "xterm*". [#]_
- The most used terminals are, for the most part, basically xterm-compatible-ish

Because of this, almost every terminal claims to be some form of "xterm" - most often ``xterm-256color``. Some people claim that this is wrong and you should simply configure the terminal to use its own terminfo entry (`e.g. <http://jdebp.uk./Softwares/nosh/guide/commands/TERM.xml>`_). I can't agree - that just makes ssh'ing awkward, breaks programs and, most importantly, would require me to tell users to reconfigure their terminal.

Which means, if I want to distinguish gnome-terminal from konsole, I need to look elsewhere - both claim to be xterm-256color by default.

But at least I can use terminfo as a baseline, right?

.. [#] There is also termcap, which is slightly different. I'm mostly familiar with terminfo so I'm gonna talk about that.
.. [#] This includes **emacs** - it has its own terminal database that entries need to be added to.
.. [#] Alacritty had its entry added to the ncurses terminfo in 2018, and in 2020 you wouldn't expect most servers to have it.
       
Terminfo is woefully incomplete or wrong
----------------------------------------

Remember that ``CursorShape`` sequence from above? There *is* an entry in terminfo for that, by the wonderfully descriptive name of "DECSCUSR" or "Ss" for short

.. code-block:: fish

  > infocmp -x | grep -o 'Ss=.*'
  Ss=\E[%p1%d q,

..but it uses the actual xterm value of ``\e[something q``, not the konsole or iTerm value of ``\e]50/1337something``.

And even if it did, you still couldn't use it, because macOS' terminfo is old and crusty and doesn't have Ss, at all.

Of course when I say "wrong" here, it's not that I'm blaming the terminfo for not matching up to a terminal it's never been written for. I'm just saying that the terminfo is, from the application's perspective, wrong. It says things that simply aren't correct. I'm not interested in finding someone to blame, I'm interested in things being broken.

Or consider the sequence to communicate the current directory to the terminal. As far as I can tell, that one is *actually defined* as being "OSC 7" [#]_. OSC is an acronym and stands for ``\e]``. [#]_

There is, as best as I can tell, no terminfo entry for it, and I'm really not expecting any.

What's more, the semantics for OSC 7 are weirdly complicated. Here's how fish handles it

.. code-block:: fish

  printf \e\]7\;file://%s%s\a $hostname (string escape --style=url $PWD)

So it starts with an escape, then a ``]7;``, then *a url to the current directory*. With actual URL-encoding. In a terminal. And then it ends with a bel character, just to make sure this wakes up anyone who uses a terminal that doesn't support it.

Or consider that it took *an actual literal decade* for terminfo to finally gain a way to say that a terminal supports 24-bit true color. [#]_

.. [#] Not that I've ever found any good documentation on any of this, mind you.
.. [#] Ackchually it stands for "Operating System Control", for some reason.
.. [#] terminfo support arrived in 2018 with ncurses 6.1. Konsole had truecolor support in **2008**.

How do we fix it?
-----------------

So if we wanted to fix this, what would have to be done?

We would need standardization, and flexible standardization at that.

My preferred solution would be a mix of having an actual baseline of support combined with making optional features just ignored by the terminal if it doesn't want to support them.

For many things, the application doesn't really *care* if it necessarily ends up being used, it just wants to not break things. For example the cursor shaping sequence should either cause the cursor to be changed or nothing to happen, so the application can just fire and forget.

Or truecolor sequences should just be used basically everywhere, in the same format (the "correct" syntax uses colons, but most terminals support it with semicolons, some do both). If the terminal is incapable of rendering truecolor, let it pick the nearest color and use that instead.

Key escapes should just be the same everywhere, and there should be standardized sequences for expressing e.g. ctrl-i as a distinct keycombination from tab.

So a terminal would set $NU_TERM to true, and it would signal to the app that the baseline is safe to use. If the app wants to have ctrl-i encoded specially, it should send ``\e]666;to-the-future!\a``, and everything should be grand and kittens will fall from the heavens (and safely land on their cute little feetsies, of course).

And people would have to stop using bad terminals.

I'm not holding my breath.
