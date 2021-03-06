Formatted Exceptions
====================

Pretty's main focus is on formatting of exceptions for readability, addressing one of Clojure's core weaknesses.

Rationale
---------


Exceptions in Clojure are extremely painful for many reasons:

* They are often nested (wrapped and rethrown)
* Stack frames reference the JVM class for Clojure functions, leaving the user to de-mangle the name back to the Clojure name
* Stack traces are output for every exception, which clogs output without providing useful detail
* Stack traces are often truncated, requiring the user to manually re-assemble the stack trace from several pieces
* Many stack frames represent implementation details of Clojure that are not relevant

This is addressed by the ``io.aviso.exception/write-exception`` function; it take an exception
and writes it to the console, ``*out*``.

This is best explained by example; here's a SQLException wrapped inside two RuntimeExceptions, and printed normally:

::

  java.lang.RuntimeException: Request handling exception
    at user$make_exception.invoke(user.clj:30)
    at user$eval1322.invoke(NO_SOURCE_FILE:1)
    at clojure.lang.Compiler.eval(Compiler.java:6619)
    at clojure.lang.Compiler.eval(Compiler.java:6582)
    at clojure.core$eval.invoke(core.clj:2852)
    at clojure.main$repl$read_eval_print__6588$fn__6591.invoke(main.clj:259)
    at clojure.main$repl$read_eval_print__6588.invoke(main.clj:259)
    at clojure.main$repl$fn__6597.invoke(main.clj:277)
    at clojure.main$repl.doInvoke(main.clj:277)
    at clojure.lang.RestFn.invoke(RestFn.java:1096)
    at clojure.tools.nrepl.middleware.interruptible_eval$evaluate$fn__808.invoke(interruptible_eval.clj:56)
    at clojure.lang.AFn.applyToHelper(AFn.java:159)
    at clojure.lang.AFn.applyTo(AFn.java:151)
    at clojure.core$apply.invoke(core.clj:617)
    at clojure.core$with_bindings_STAR_.doInvoke(core.clj:1788)
    at clojure.lang.RestFn.invoke(RestFn.java:425)
    at clojure.tools.nrepl.middleware.interruptible_eval$evaluate.invoke(interruptible_eval.clj:41)
    at clojure.tools.nrepl.middleware.interruptible_eval$interruptible_eval$fn__849$fn__852.invoke(interruptible_eval.clj:171)
    at clojure.core$comp$fn__4154.invoke(core.clj:2330)
    at clojure.tools.nrepl.middleware.interruptible_eval$run_next$fn__842.invoke(interruptible_eval.clj:138)
    at clojure.lang.AFn.run(AFn.java:24)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1110)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:603)
    at java.lang.Thread.run(Thread.java:722)
  Caused by: java.lang.RuntimeException: Failure updating row
    at user$update_row.invoke(user.clj:22)
    ... 24 more
  Caused by: java.sql.SQLException: Database failure
  SELECT FOO, BAR, BAZ
  FROM GNIP
  failed with ABC123
    at user$jdbc_update.invoke(user.clj:6)
    at user$make_jdbc_update_worker$reify__214.do_work(user.clj:17)
    ... 25 more

On a good day, the exception messages will include all the details you need to resolve the problem ... even though
Clojure encourages you to use the ``ex-info`` to create an exception,
which puts important data into properties of the exception, which are not normally printed.

Meanwhile, you will have to mentally scan and parse the above text explosion, to parse out file names and line numbers,
and to work backwards from mangled Java names to Clojure names.

It's one more bit of cognitive load you just don't need in your day.

Instead, here's the equivalent, using a *hooked* version of Clojure's ``clojure.repl/pst``,
modified to use ``write-exception``.

.. image:: images/formatted-exception.png
   :alt: Formatted Exception

As you can see, this lets you focus in on the exact cause and location of your problem.

``write-exception`` flips around the traditional order, providing a chronologically sequential view:

* The stack trace leading to the root exception comes first, and is ordered outermost frame to innermost frame.

* The exception stack comes after the stack trace, and is ordered root exception (innermost) to outermost, reflecting how the
  stack has unwound, and the root exception was wrapped in new exceptions and rethrown.

The stack trace is carefully formatted for readability, with the left-most column identifying Clojure functions
or Java class and method, and the right columns presenting the file name and line number.

The stack frames themselves are filtered to remove details that are not relevant.
This filtering is via an optional function, so you can define filters that make sense for your code.
For example, the default filter omits frames in the clojure.lang package (they are reduced to ellipses), and truncates the
stack trace when when it reaches clojure.main/repl/read-eval-print.

Repeating stack frames are also identified and reduced to a single line (that identifies the number of frames).
This allows your infinite loop that terminates with a StackOverflowException to be reported in just a few lines, not
thousands.

The inverted (from Java norms) ordering has several benefits:

* Chronological order is maintained, whereas a Java stack trace is in reverse chronological order.

* The most relevant details are at (or near) the *bottom* not the *top*; this means less "scrolling back to see what happened".

The related function, ``format-exception``, produces the same output, but returns it as a string.

For both ``format-exception`` and ``write-exception``, output of the stack trace is optional, or can be limited to a certain number of stack frames.

io.aviso.repl
-------------

This namespace includes a function, ``install-pretty-exceptions``, which
hooks into all the common ways that exceptions are output in Clojure and redirects them to use write-exception.

When exceptions occur, they are printed out without a stack trace or properties.
The ``clojure.repl/pst`` function is overridden to fully print the exception (*with* properties and stack trace).

In addition, ``clojure.stacktrace/print-stack-trace`` and ``clojure.stacktrace/print-cause-trace`` are overwritten; these
are used by ``clojure.test``. Both do the same thing: print out the full exception (again,
with properties and stack trace).

You may not need to invoke this directly, as
pretty can also act as a :doc:`lein-plugin`.

io.aviso.logging
----------------

This namespace includes functions to change ``clojure.tools.logging`` to use Pretty to output exceptions, and to add a
default Thread.UncaughtExceptionHandler that uses ``clojure.tools.logging``.