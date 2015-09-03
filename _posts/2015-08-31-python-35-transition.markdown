---
layout: post
status: publish
title: Homebrew Python 3.5 transition
---

Python 3.5 is [scheduled to be released](https://www.python.org/dev/peps/pep-0478/) on September 13, 2015. Since Homebrew carries the newest stable version of its packages, the Homebrew `python3` formula will adopt Python 3.5 very quickly upon its release. This post explains how migrating to Python 3.5 will affect Homebrew users who have set up a Python 3 development environment using Homebrew packages.

## Packages installed with pip will not be found and scripts will break

Python packages which were installed against Python 3.4 with pip or easy\_install will need to be reinstalled for Python 3.5. Packages installed against Python 3.4 will not be visible from Python 3.5 because Python uses versioned site-packages directories. Scripts (i.e. `ipython3`) which were installed with pip or easy\_install using Python 3.4 will stop working until they are reinstalled.

To reconstitute your Python environment, consider running `pip3 freeze > site-requirements.txt` before you upgrade to save a list of your installed packages. You should edit this list by hand to remove any packages which are managed by Homebrew. Then, you can run `pip3 -r site-requirements.txt` after you upgrade to reinstall your environment.

(If you forget to do this before you upgrade, you can run `$(brew --cellar)/python3/3.4.3_2/bin/python3 -m pip freeze` to generate the list of packages 3.4 knows about, even after 3.5 is installed.)

## Homebrew packages built with `--with-python3` will break

Homebrew will take care of rebuilding the relatively few Homebrew packages which have mandatory or recommended dependencies on `"python3"` or `:python3`. If you are only using these packages, you won't need to do anything. (The complete list of those packages is: circlator, gubbins, iva, keepassc, lensfun, libsigrokdecode, mypy, pastebinit, ponysay, pulseview, py3cairo, pymummer, pyqt5, retext, and xonsh.)

Packages which may be *optionally* built against python3 by passing `--with-python3` will break and will need to be reinstalled by you. You can get a list of packages which were installed using `--with-python3` using brew and jq like:

`brew info --json=v1 --installed | jq -r '.[] | if (.installed | map(.used_options | any(contains("with-python3"))) | any) then .full_name else empty end'`

Running `brew reinstall` for each of those packages will bring your system up to date after the transition. (You can equivalently pipe the output into `xargs brew reinstall`.)

## Continuing to use Python 3.4

Life on the bleeding edge isn't for everyone. If you would prefer to continue using Python 3.4, please consider uninstalling the python3 formula and using [pyenv](https://github.com/yyuu/pyenv) instead. You can install pyenv with Homebrew with `brew install pyenv`. Pyenv is a great tool for managing multiple parallel Python installations.

I prefer not to accept Python versions into homebrew-versions because of the relatively high maintenance load, but [starting a tap](https://github.com/Homebrew/homebrew/blob/master/share/doc/homebrew/How-to-Create-and-Maintain-a-Tap.md) (Homebrew's term for an alternative formula repository) is easy to do if you'd like to maintain a python34 formula yourself (or a suite of similar formulas, like Felix Krull's [deadsnakes](https://launchpad.net/~fkrull/+archive/ubuntu/deadsnakes) PPA for Ubuntu). I would owe you a beer. :)

## Migrating early

If you would like to transition to Python 3.5 before its official release, you can run `brew install python3 --devel` to install the release candidate. After 3.5 final is released, a normal `brew update && brew upgrade` will upgrade you to the release version. Anything that works with a pre-release version of 3.5 should continue working with a release version of 3.5.

## Questions

If you need help, please feel free to reach out on Homebrew's Github issue tracker or the mailing list (homebrew@librelist.com). Thanks for your patience!
