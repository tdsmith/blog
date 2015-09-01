---
title: Building Python extension modules on OS X with cmake or autotools
layout: post
status: publish
---

tl;dr: Please use `-undefined dynamic_lookup` instead of `-lpython` or `-framework Python` to build Python extension modules on OS X, no matter what `python-config` says. Only use `-lpython` or `-framework Python` if you intend to embed an interpreter.

When a project with a cmake or autotools-based build system decides to start shipping Python bindings, developers often notice that the default behavior of the Apple and Linux linkers is different. On Linux, building with `-shared` tells the linker to ignore undefined symbols, assuming that the dynamic linker will be able to resolve them at runtime. A consequence is that it's unnecessary to pass `-lpython` to the linker when building extension modules.

On OS X, using `-shared` (or, more correctly, `-bundle`) on its own will still result in the linker demanding that all symbols are resolved at link time. Some developers work around this by adding `-lpython` or `-framework Python` to their linker invocations. (Sometimes, this is done implicitly by trusting the output of `python-config --ldflags`.) This appears to work but it is not the best solution. The best solution is to provide `-bundle -undefined dynamic_lookup`, which has the same behavior as the Linux linker.

## Why -lpython is wrong

The effect of `-lpython` or `-framework Python` is to unnecessarily force the extension module to be used with the Python interpreter against which it was built. Building a module against one Python interpreter and then attempting to import it from a Python interpreter installed to a different location will cause the interpreter to segfault even if the build and import interpreters are ABI compatible. This often appears in practice if someone builds bindings with system Python and then later installs a compatible version of Python from another source (Homebrew, Python.org, pyenv) and tries to import the already-built bindings using their new python. It may also manifest if the first python in `$PATH` is not what a user thinks it should be during their build. It is a common source of [crash reports](https://github.com/Homebrew/homebrew/search?utf8=%E2%9C%93&q=PyThreadState_Get+OR+%28python+AND+%22Segmentation+fault%22%29&type=Issues) to projects and to Homebrew, often leading to user and developer frustration and hairy workarounds. 

Another consequence is that the compiled module cannot be redistributed in a wheel or bdist -- or in Homebrew's bottles -- because those must be portable to different installation locations.

You can detect if a module is explicitly linked to a Python library by running `otool -L my_module.so`, which will reveal any links to a python library or framework binary. For example, you may see `/System/Library/Frameworks/Python.framework/Versions/2.7/Python` for a module linked explicitly to system Python. If your module is built correctly, you should see no reference to Python at all.

## What to do instead

If you are building an extension module, pass `-undefined dynamic_lookup` instead of `-lpython` or `-framework Python`. If you are worried that the linker will stop complaining about other symbols which may be missing, you can optionally use the `-bundle_loader` flag, which asks the linker to check that the specified library provides any missing symbols, but does not link to it.

Note that you *do* need to pass `-lpython` or `-framework Python` when linking something that embeds the Python interpreter, but those things are probably not `import`able.

You should not trust the output of `python-config --ldflags` or `--libs`, which give the right answer for embedding an interpreter, but not for building extension modules. (In fact, the respective intentions of python-config and python's pkgconfig files [are not well-defined](https://bugs.python.org/issue15590).)

Indeed, `-undefined dynamic_lookup` is how Python's distutils builds extensions (run `pip install -vU --force-reinstall --no-use-wheel markupsafe` for proof). I've helped several projects, including Boost.Python, opencv, pycairo, openimageio, and ledger, change their build systems to match this behavior. Afterwards, these projects and Homebrew see many fewer crash reports from users on OS X.

If you have any questions about this easy way to safely improve the portability of your bindings on OS X, please reach out. I'm tdsmith in #machomebrew and #pypa on Freenode.
