Sometimes, support tools can delight
####################################

:date: 2019-06-28 19:20
:category: fish

What's this blogging thing all about? Lemme check!

So, there's a recent addition to fish_ that I really like. It's called "littlecheck", and it's a new test driver [#]_.

Now, test drivers are unlikely to ever really receive praise (or even be mentioned at all in most contexts), but they're surprisingly impactful. A bad test driver will leave you scratching your head if you have a failure, or will introduce friction to adding tests, which will make people not add tests.

For example, in our previous driver, each test consisted of three files:

- test.in, the file to actually run.
- test.out - the stdout of running test.in
- test.err - the stderr

Which had a few problems:

- To create a new test, you need to create 3 files.
- To figure out where a test failed, you need to follow 3 files at the same time
- The ".in" is a fish script, but it can't be called ".fish" for internal reasons [#]_

The replacement looks a little something like this:

.. code-block:: fish

    #RUN: %fish %s
    echo banana
    #CHECK: banana
    date +%Y
    #CHECK: 20{{\d+}}

Which immediately solves all the above issues (the files are indeed called "something.fish" [#]_), but also some I haven't even talked about yet.

See, we had an odd bug where redirecting to a directory (``echo something > /directory/``) didn't immediately fail. That's certainly something you'd like to test, and we did for quite a while. We just tested that redirecting to `.` failed with an error, and that was that. One file for the command, one file for the error and one file just because.

Fast forward 5 years to this January, when I was working on getting test suite to succeed on Solaris [#]_. As it turns out Solaris also prints an error, but a different one (instead of "Is a directory" EISDIR you'd get "Invalid argument" EINVAL). The old test driver had no way to express this, and the error couldn't be redirected (because it's a syntax error), so this required us to disable the test. [#]_

Then we got another test driver (yes, that's two in parallel), which ran "invocation tests", which checked how fish behaved when calling the binary with options. This was enough of an indirection that we managed to wedge in a way to use different test output depending on `uname`, so it worked again. But this required:

- A second test driver
- Checking operating system based on _name_, not feature testing. Using `uname` should be avoided if at all possible.

So what does littlecheck do here? See that ``{{\d+}}`` above? That's an embedded regex that matches any string of digits, meaning that ``date`` test will succeed as long as it's run in this century, or in about 18000 years [#]_. Or, alternatively, you could use it for something useful like this actual test:

.. code-block:: fish

    #RUN: %fish %s
    begin; end > . ; status -b; and echo "status -b returned true after bad redirect on a begin block"
    # Note that we sometimes get fancy quotation marks here, so let's match three characters
    #CHECKERR: <W> fish: An error occurred while redirecting file {{...}}
    #CHECKERR: {{open: Is a directory|open: Invalid argument}}
    echo $status
    #CHECK: 1

So you just embed the possibilities in a regex, and it'll just allow both. [#]_

Hopefully you'll see why I really like this (thanks ridiculousfish!), and I haven't even mentioned the cherries on top:

- It's entirely fish-agnostic - this can run any scripting language (that uses `#` as a comment character)
- It's one single python file, no further dependencies
- It's public domain [#]_

Littlecheck is available at https://github.com/ridiculousfish/littlecheck. I kinda love it right now.

.. [#] I define "test driver" as "thing that runs the tests".
.. [#] We glob, and they are in the same directory as the main "test.fish" orchestration script. I didn't say it was a good reason!
.. [#] Which makes text editors highlight them as fish scripts, which makes me happy.
.. [#] Or Illumos, or OpenIndiana? I still don't get the nomenclature.
.. [#] Alternatively, we could have caught the error and swapped it for the other one. But that would require behavioral changes, and it's not clear that every EINVAL is because of a directory, so we'd have to figure that out or just use the more generic error everywhere.
.. [#] It's not an actual test we use.
.. [#] For astute readers, there's another thing this allows, which required us to disable even the invocation tests.
.. [#] My thoughts on licensing are a tad more complex, but in short I believe for a simple thing like this you want it to be drop-in-and-forget.
.. _fish: https://fishshell.com
