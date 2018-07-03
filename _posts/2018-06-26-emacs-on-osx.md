---
title: Emacs on MacOS
layout: post
tags:
    - emacs
    - macos
keywords:
    - emacs
    - macos
---


After almost 15 years of running emacs predominantly on linux in
terminal mode, I am planning on switching my workflow completely to
MacOS. At the same time, I have experimented with the capabilities of
emacs running in a window/graphical system instead of the terminal
interface. Both experiences have been pleasant so far, particularly
the fact that I can now view images in emacs buffers :-O However,
there have been some issues that I have had to address, so here it
goes.

## Operating emacs in a window-system

A lot of flows in emacs depend on things like environmental variables
being set correctly. My current bash setup involves a `.profile` file
that specifies stuff like the `PATH` and `GOPATH`, `AWS_*` variables,
various aliases, shell customizations (such as `PS1`) as well as
`rvm`-related functionality. The `.profile` file is sourced in
`.bashrc`, so a login shell is needed to get all these declarations in
the environment.

One of the first things I noticed when running emacs on the MacOS
window-system (`C-h v window-system`) was the fact that the
environment in emacs was pretty much void of all my
customizations. So, for example the `projectile-compile-project`
command in a go project failed with errors indicating that the
environment is not setup correctly (`GOPATH` not defined, `dep`
located in `/usr/local/bin` was not found etc.).

And indeed it makes sense for the environment to be void; why should
it be otherwise? When the application starts in a window system, there
is no reason why the custom bash initialization should be executed in
the context of the application. This should be true in all window
systems, not just MacOS (`ns`).

It turns out that this problem can be solved using the
[`exec-path-from-shell`](https://github.com/purcell/exec-path-from-shell)
package.
TODO ADD LINK HERE AND DESCRIBE PACKAGE

Add the following in your emacs initialization file to get the package
working:

```lisp
;; this will make sure that the package is installed during emacs init
(use-package exec-path-from-shell
  :ensure t)

;; this will initialize the package only when a window-system is detected
(when (memq window-system '(mac ns x))
  (exec-path-from-shell-initialize))
```

## Supporting multiple system-types in `init.el`

Now, going back to the issue of switching emacs usage from linux to
MacOS, I would very much like to support both systems in a single
`init.el` because some usage on linux is to be expected after all. For
functionality that differs across the two systems, emacs provides the
variable `system-type` (`C-h v system-type`).

For example, to use the `badger` theme on MacOS only one needs to do
the following in `init.el`:

```lisp
(if (eq system-type 'darwin)
    (load-theme 'badger t))
```

So, using the `system-type` variable, one can execute lisp code
conditionally upon the type of the system in which emacs is running.
