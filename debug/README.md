# Debugging Python code

This is a tutorial how to get started with debugging Python code by Christoph
Deil.

We will start by looking at how Python executes code, exceptions and stack
frames first, and only in the second half move on to using a debugger.

This tutorial assumes that you have used a terminal, Python, ipython and Jupyter
before. No experience with Python debugging is assumed, this tutorial will get
you started and focus on the basics.

Throughout the tutorial you will find short exercises marked with :point_right:.
Usually the solution is given directly below. Please execute the examples and
try things for yourself. Interrupt with questions at any time!

This is the first time I'm giving a tutorial on this topic. Please let me know
if you have any suggestions to improve!

## Outline

- [Setup](#setup)
- [1. When to debug?](#1-when-to-debug)
- [2. How Python executes code](#2-how-python-executes-code)
- [3. Exceptions and tracebacks](#3-exceptions-and-tracebacks)
- [4. Debugging with pdb](#4-debugging-with-pdb)
- [5. Debugging with ipython](#5-debugging-with-ipython)
- [6. Debugging with Jupyter](#6-debugging-with-jupyter)
- [7. Debugging with PyCharm](#7-debugging-with-pycharm)
- [Things to remember](#things-to-remember)
- [Going further](#going-further)

## Questions

Please help me adjust the tutorial content and speed a bit:

- How often do you debug Python code? (never, last year, all the time)?
- Do you know how Python executes code?
- Do you know what exceptions and stack frames are and how to read a traceback?
- Have you used `pdb` to debug from Python? From pytest?
- Have you used `%debug` or `%run -d` from ipython or Jupyter?
- Have you used the `PyCharm` visual debugger? Or debugged from any other IDE (e.g. VS Code or emacs)?
- Have you used any other Python debugging tool?

## Setup

:point_right: Check that you have Python (3.5 or later), `ipython` and `jupyter`
installed.

```
$ python --version
$ ipython --version
$ jupyter --version
```

If you don't have this, one nice option to get it is [Anaconda
Python](https://www.anaconda.com/download/).

---

At the end of this tutorial, i will demo how to use the visual debugger in
PyCharm.

If you want to try it out, install the free community edition of
[PyCharm](https://www.jetbrains.com/pycharm/download/). After installing
PyCharm, you need to configure two things: your Python interpreter and execute
`Tools | Create Command-line Launcher`.

:point_right: Check that you have PyCharm installed and configured.

One way to launch PyCharm is to cd into the directory for this tutorial and use
the command line launcher like this:
```
cd School2019/debug
charm .
```
Then right-click on `analysis.py` and select "run analysis". A console at the
bottom should appear the output of "5.0" that we print from that script.

*Note: there are many other editors and IDEs that have Python debugging support
(either built in or via extensions), e.g. `vim` or `emacs` or [Visual Studio
Code](https://code.visualstudio.com/). I'm not familiar with those, and in any
case we will not have time to sort out installation / setup problems for those
during the tutorial. If you want to use those, try them after the tutorial and
try to re-do the examples from this tutorial.*

---

As part of this tutorial, we will go over the [Errors and Debugging](https://jakevdp.github.io/PythonDataScienceHandbook/01.06-errors-and-debugging.html)
Jupyter notebook from the excellent [Python Data Science
Handbook](https://jakevdp.github.io/PythonDataScienceHandbook/) by Jake
VanderPlas. It's freely available at
https://github.com/jakevdp/PythonDataScienceHandbook and generally is a great
resource to learn, so I wanted to introduce it.

:point_right: Get set up with the Python Data Science Handbook to execute the
notebooks on your computer now.

Follow these steps:
* Open a new terminal (because we'll run `jupyter lab` there and then it can't
  be used for anything else)
* Change directory to where you have your repositories
* Run these commands:
  ```
  conda activate school18
  git clone https://github.com/jakevdp/PythonDataScienceHandbook.git
  cd PythonDataScienceHandbook/notebooks
  jupyter lab Index.ipynb
  ```
* Open the "Errors and Debugging" (`01.06-Errors-and-Debugging.ipynb`)
  notebook from Chapter 1.
* Leave it open, but go back to this tutorial for now.

---

:point_right: Optional: If you have a Google account and want to try out "Google Colab", click the button [here](https://jakevdp.github.io/PythonDataScienceHandbook/01.06-errors-and-debugging.html).

From the [Google Colaboratory FAQ](https://research.google.com/colaboratory/faq.html):
> Colaboratory is a research tool for machine learning education and research.
> It’s a Jupyter notebook environment that requires no setup to use.

To open your space: https://colab.research.google.com/

## 1. When to debug?

### Suspect result

When you have an incorrect or at least suspect output of your program, you have
to investigate your code and data to try and pin down why the output is not what
you expect. This is the worst, compared to this issue, the next two cases are
nice, because it's obvious that there's a problem and you get a traceback with
lots of info where to start looking.

### Exception

Most of the time you will be able to read the traceback and code and figure out
what is wrong and not need to start a debugger. But sometimes it's not clear and
you need to 'look around'; that's when you start a debugger.

### Crash

The Python process can crash. This is very rare, except if you work on or use
buggy Python C extensions. To debug it you would use `gdb` or `lldb`. There are
tutorials (see e.g.
[here](http://www.scipy-lectures.org/advanced/debugging/index.html#debugging-segmentation-faults-using-gdb)),
we won't cover it here.

:point_right: Cause Python to crash.

```
$ python
>>> import ctypes
>>> ctypes.string_at(1)
Segmentation fault: 11
$
```

## 2. How Python executes code

To debug Python code, you need to know how Python executes code. Have a look at
the Python module [point.py](point.py) that defines a `Point` class and a
`distance` function, and the [analysis.py](analysis.py) script that does `from
point import Point, distance` and runs a simple analysis.

Is it clear what happens when you run `python analysis.py`?

The short answer is that Python executes code top to bottom, line by line. When
a `def` or `class` statement is encountered, a function or `type` object are
created in the module `namespace`, and `import point` causes the code in
`point.py` to be executed, and when the bottom is reached, the `point` module is
stored in the global `sys.modules` dict, i.e. a second `import point` will be a
no-op, not execute the code in `point.py` again. You should never reload in
Python, always restart the interpreter if you edit any code.

If you're not sure what Python does with `def` or `class` or `import`, please
ask now, and we'll spend a few minutes to add print statements to show what is
going on.

Another important concept you need to know about is how Python variables and
function calls work. Superficially Python seems similar to C or C++, there are
variables to store data and function calls create stack frames. But if you look
a bit closer, you'll see that it works completely differently under the hood: in
Python everything is an object, variables are entries in namespace dictionaries
(`globals()` and `locals()`) pointing to objects, and Python is dynamic, i.e.
happy to have an integer variable `data = 999` and then on the next line change
to a string variable `data = 'spam'`. Memory management is automatic, using a
reference counting garbage collector that deletes objects with zero references.
Python is both "compiled" and "interpreted": code is parsed into an `ast`
(abstract syntax tree), compiled into `bytecode`, and executed by the `CPython`
"interpreter" or "virtual machine", which executes one byte code after the other
in an infinite `while` loop.

Most Python programmers don't know how Python works "under the hood". That's
good, Python is supposed to be a high-level language that "fits your brain" and
does what you intuitively expect. But you still need to have a "mental model"
about variables, objects and the stack of function calls, each with it's own
local namespace. The best way to learn about this is actually to step through
code and see how the Python program state changes. Before we do this in a
debugger, check this out:

:point_right: Step through the [point example](https://goo.gl/yTEbLX) using
http://pythontutor.com/.

If you'd like to learn more, the [Whirlwind Tour Of
Python](http://nbviewer.jupyter.org/github/jakevdp/WhirlwindTourOfPython/blob/master/Index.ipynb)
is a beginner-level introduction, and [Python
epiphanies](https://nbviewer.jupyter.org/github/oreillymedia/python_epiphanies/blob/master/Python-Epiphanies-All.ipynb)
is a notebook explaining everything in great detail.

## 3. Exceptions and Tracebacks

If you use Python, you will see exceptions and tracebacks all the time.

In Python, the term "error" is often used to mean the same thing as "exception".
Although of course, "error" and "exception" are very general terms and are
widely used, not always referring to a Python exception (i.e. instances of the
built-in `Exception` class or subclasses of `Exception` like `TypeError`).

Once you've learned to read a traceback, there will be many times where it's
enough information for you to figure out what's wrong, and you will only start
the debugger in the cases where the problem isn't clear from reading the
traceback and relevant code for a minute or two.

Example: [exception.py](exception.py)
```python
$ python exception.py
Traceback (most recent call last):
  File "exception.py", line 14, in <module>
    main()
  File "exception.py", line 12, in main
    move_it(p)
  File "exception.py", line 8, in move_it
    point.move(42, '43')
  File "/Users/deil/code/python-tutorials/debug/point.py", line 16, in move
    self.y += dy
TypeError: unsupported operand type(s) for +=: 'int' and 'str'
```

Sometimes you will see a "chained exception" (also called "double inception"),
where a second exception is raised inside an exception handler, i.e. an `except`
block.

Example: [exception_chain.py](exception_chain.py)
```
$ python exception_chain.py 
Traceback (most recent call last):
  File "exception_chain.py", line 4, in <module>
    result = a / b
ZeroDivisionError: division by zero

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "exception_chain.py", line 6, in <module>
    result = c
NameError: name 'c' is not defined
```

:point_right: Start `python` and create some of the most common exceptions.

These are some very common errors you'll see a lot:
- `SyntaxError`
- `IndentationError`
- `NameError`
- `AttributeError`
- `KeyError`
- `IndexError`

:point_right: What other exceptions have you seen? Are they clear or do you have
any question?

There's more info on Python exceptions
[here](https://docs.python.org/3/tutorial/errors.html) and an overview of all
built-in exceptions
[here](https://docs.python.org/3/library/exceptions.html#exception-hierarchy).

See also the Zen of Python (try: `import this`) and "Errors should never pass
silently."

## 4. Debugging with pdb

:point_right: Learn to debug using `pdb`. Follow along!

We will use [exception.py](exception.py) and [silent_error.py](silent_error.py) as examples.

Tasks:

- Run `python exception.py` and read the traceback
- Add print statements in the method that fails (`point.py` line 18)
- Add `import pdb; pdb.set_trace()` in the method that fails and learn the
  debugger commands: `p(rint)`, `w(here)`, `l(ist)`, `ll` or `longlist`,
  `h(elp)`, `q(uit)`, `n(ext)`, `u(p)`, `d(own)`
- Run `python -m pdb exception.py` and type `c(ontinue)` to enter the debugger
  and jump to the point of exception ("post mortem debugging").
- To debug tests, use the `--pdb` option of `pytest`. Run `pytest --pdb
  exception.py` to see an example (not really a test, but it's the same for a
  failing test).
- Debug `silent_error.py` by setting a breakpoint in `compute_result`.

```
$ python -m pdb silent_error.py 
> /Users/deil/learn/School2019/debug/silent_error.py(2)<module>()
-> """
(Pdb) b compute_result
Breakpoint 1 at /Users/deil/learn/School2019/debug/silent_error.py:12
(Pdb) c
> /Users/deil/learn/School2019/debug/silent_error.py(15)compute_result()
-> val = 2 * data["a"]
(Pdb) p data
{'a': 2, 'b': '3'}
(Pdb) p type(data['b'])
<class 'str'>
(Pdb) q
```

:point_right: Who is using pytest?

It's great, you should. The time you spend to write automated tests pays off
pretty quickly (fewer errors, less testing and debugging). Use [pytest](https://docs.pytest.org)
and learn about it using the tutorial [here](https://github.com/jiffyclub/pytest-features).

If you want to learn more about `pdb`, go through a detailed tutorial at your
own pace:
- The [Python Debugging With Pdb](https://realpython.com/python-debugging-pdb/)
  tutorial by Nathan Jennings.
- The Python standard library documentation for `pdb`
  ([link](https://docs.python.org/3.6/library/pdb.html))
- The Python module of the week tutorial for `pdb` by Doug Hellman
  ([link](https://pymotw.com/3/pdb/index.html))

## 5. Debugging with ipython

Similarly how `ipython` and `jupyter` often give a nicer interactive Python
environment than `python`, they also make it often easier to debug.

Try this:

- `ipython -i` to execute a script and jump into IPython
- `ipython --pdb` to execute a script and jump into ipydb
- Add `import IPython; IPython.embed()` to drop into IPython on a given line

More examples in the next session, from Jupyter, but it's similar from ipython.

## 6. Debugging with Jupyter

To learn debugging from Jupyter, let's use the [Errors and
Debugging](https://jakevdp.github.io/PythonDataScienceHandbook/01.06-errors-and-debugging.html)
notebook from the the excellent [Python Data Science
Handbook](https://jakevdp.github.io/PythonDataScienceHandbook/) by Jake
VanderPlas. It's freely available at
https://github.com/jakevdp/PythonDataScienceHandbook and generally is a great
resource to learn, so I wanted to introduce it.

:point_right: Clone the `PythonDataScienceHandbook` git repository and start
Jupyter.
```
cd <where you have your repositories>
git clone https://github.com/jakevdp/PythonDataScienceHandbook.git
cd PythonDataScienceHandbook
jupyter notebook notebooks/01.07-Timing-and-Profiling.ipynb
```

- `%debug`
- `%run -d`
- `%pdb`

## 7. Debugging with PyCharm

PyCharm has a great visual debugger. 

Since PyCharm is visual, it's hard to write a tutorial so that you can follow
along offline by just reading the info here. Note that the PyCharm folks have a
tutorial on debugging with many screenshots
[here](https://www.jetbrains.com/help/pycharm/debugging-code.html) and a 6 min
video on Youtube [here](https://www.youtube.com/watch?v=QJtWxm12Eo0).

One way to launch PyCharm is to cd into the directory for this tutorial and use
the command line launcher like this:
```
cd python-tutorials/debug
charm .
```

:point_right: Right-click on `analysis.py` and select "debug analysis".

## Things to remember

- Python is a very dynamic language. It's very powerful, easy to make mistakes,
  and easy to inspect and debug.
- Debugging is not something you do all the time, you should aim to do it as
  little as possible. Learn how Python executes code and how to read tracebacks.
  Write clean, well-structured code (functions and classes) and automated tests.
- Learn how to debug from various execution environments: Python, pytest,
  ipython, Jupyter, PyCharm. If you only know how to debug one way, you'll have
  to change execution environment and waste time: e.g. if you get an error in a
  test, you don't want to have to start Jupyter and reproduce the issue there
  before you can debug it.
- Use `pdb` from Python and `ipdb` from ipython and Jupyter for debugging, or a
  visual debugger like e.g. the one from Pycharm.
- See the debugger commands with `help` or
  [here](https://docs.python.org/3.6/library/pdb.html#debugger-commands).
- From ipython / jupyter, the commands are `%debug`, `%run -d` and `%pdb`
- Most people don't use a debugger often. There's code reading and `print` and
  `IPython.embed()` or just using ipython and the Jupyter notebook to see what's
  going on.

## Going further

These are good resources to learn more:

- Use http://pythontutor.com/ to learn how Python executes code.
- The [Errors and
  Debugging](https://jakevdp.github.io/PythonDataScienceHandbook/01.06-errors-and-debugging.html)
  page from the [Python data science
  handbook](https://jakevdp.github.io/PythonDataScienceHandbook/) by Jake
  VanderPlas.
- The Python standard library documentation for `pdb`
  ([link](https://docs.python.org/3.6/library/pdb.html))
- The Python module of the week tutorial for `pdb` by Doug Hellman
  ([link](https://pymotw.com/3/pdb/index.html))
- The [Python Debugging With Pdb](https://realpython.com/python-debugging-pdb/)
  tutorial by Nathan Jennings.
- The [Machete mode debugging](https://nedbatchelder.com/text/machete.html) talk
  by Ned Batchelder is fun and educational.
- [Software debugging](https://eu.udacity.com/course/software-debugging--cs259)
  free course on Udacity by Andreas Zeller. (I only watched the 1 min intro,
  don't know if it's any good.)
