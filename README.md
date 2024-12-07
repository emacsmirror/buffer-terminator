# buffer-terminator.el - Terminate Inactive Emacs Buffers Automatically
![Build Status](https://github.com/jamescherti/buffer-terminator.el/actions/workflows/ci.yml/badge.svg)
![License](https://img.shields.io/github/license/jamescherti/buffer-terminator.el)
![](https://raw.githubusercontent.com/jamescherti/buffer-terminator.el/main/.images/made-for-gnu-emacs.svg)

The **buffer-terminator** package automatically terminates inactive buffers to help maintain a clean and efficient workspace, while also improving Emacs' performance by reducing the number of open buffers, thereby decreasing the number of active modes, timers, and other processes associated with those inactive buffers.

By default, all inactive buffers are terminated except special buffers (special buffers are buffers whose names start with a space, start and end with `*`, or whose major mode is derived from `special-mode`).

When a buffer is not a special buffer (e.g., a file-visiting or dired buffer), only buffers that have been inactive for a specified period are terminated. (Exception: modified file-visiting buffers that have not been saved are not terminated; the user must save them first.)

(`(buffer-terminator-mode)` terminates all the buffers that have been inactive for longer than the duration specified by `buffer-terminator-inactivity-timeout` (Default: 30 minutes). It checks every `buffer-terminator-interval` - Default: 10 minutes - to determine if a buffer should be terminated.)

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
## Table of Contents

- [buffer-terminator.el - Terminate Inactive Emacs Buffers Automatically](#buffer-terminatorel---terminate-inactive-emacs-buffers-automatically)
  - [Features](#features)
  - [Installation](#installation)
    - [Install with straight (Emacs version < 30)](#install-with-straight-emacs-version--30)
    - [Installing with use-package and :vc (Built-in feature in Emacs version >= 30)](#installing-with-use-package-and-vc-built-in-feature-in-emacs-version--30)
  - [Configuration](#configuration)
    - [Verbose Mode](#verbose-mode)
    - [Timeout for Inactivity](#timeout-for-inactivity)
    - [Cleanup Interval](#cleanup-interval)
    - [Rules](#rules)
    - [Predicate function](#predicate-function)
  - [Frequently asked questions](#frequently-asked-questions)
    - [Why? What problem is this aiming to solve?](#why-what-problem-is-this-aiming-to-solve)
    - [How is this different from the builtin midnight-mode?](#how-is-this-different-from-the-builtin-midnight-mode)
  - [Author and License](#author-and-license)
  - [Links](#links)

<!-- markdown-toc end -->

## Features

- Automatically terminates/kills inactive buffers based on a configurable timeout.
- Excludes specific buffers using buffer names, regular expressions, or special buffer types.
- Verbose mode for logging terminated buffers.
- Supports customizable intervals for periodic cleanup.
- Ensures special buffers and important user-defined buffers are preserved.

## Installation

### Install with straight (Emacs version < 30)

To install `buffer-terminator` with `straight.el`:

1. It if hasn't already been done, [add the straight.el bootstrap code](https://github.com/radian-software/straight.el?tab=readme-ov-file#getting-started) to your init file.
2. Add the following code to the Emacs init file:
```emacs-lisp
(use-package buffer-terminator
  :ensure t
  :straight (buffer-terminator
             :type git
             :host github
             :repo "jamescherti/buffer-terminator.el")
  :custom
  (buffer-terminator-verbose nil)
  :config
  (buffer-terminator-mode 1))
```

### Installing with use-package and :vc (Built-in feature in Emacs version >= 30)

To install `buffer-terminator` with `use-package` and `:vc` (Emacs >= 30):

``` emacs-lisp
(use-package buffer-terminator
  :ensure t
  :vc (:url "https://github.com/jamescherti/buffer-terminator.el"
       :rev :newest)
  :custom
  (buffer-terminator-verbose nil)
  :config
  (buffer-terminator-mode 1))
```

## Configuration

### Verbose Mode

Enable verbose mode to log buffer cleanup events:

```elisp
(setq buffer-terminator-verbose t)
```

### Timeout for Inactivity

Set the inactivity timeout (in seconds) after which buffers are considered inactive (default is 30 minutes):

```elisp
(setq buffer-terminator-inactivity-timeout (* 30 60)) ; 30 minutes
```

### Cleanup Interval

Define how frequently the cleanup process should run (default is every 10 minutes):

```elisp
(customize-set-variable 'buffer-terminator-interval (* 10 60)) ; 10 minutes
```

(Using `customize-set-variable` allows `buffer-terminator-interval` to update the timer dynamically, without the need to restart `buffer-terminator-mode`.)

### Rules

By default, *buffer-terminator* automatically determines which buffers are safe to terminate.

However, if you need to define specific rules for keeping or terminating certain buffers, you can configure them using `buffer-terminator-rules`.

The `buffer-terminator-rules` `defcustom` is a centralized defcustom that holds instructions for keeping or terminating buffers based on their names or regular expressions. Each rule is a cons cell where the key is a symbol indicating the rule type, and the value is either string or a list of strings:
```elisp
(setq buffer-terminator-rules-alist
      '((kill-buffer-name . ("temporary-buffer-name1"
                             "temporary-buffer-name2"))

        (keep-buffer-name . ("important-buffer-name1"
                             "important-buffer-name2"))

        ;; Users can also define strings
        (kill-buffer-name . "temporary-buffer-name3")

        (keep-buffer-name-regexp . ("\\` \\*Minibuf-[0-9]+\\*\\'"))
        (kill-buffer-name-regexp . "compile-angel")

        ;; Retain special buffers (Important).
        ;; Keep this in the end, after all the rules above.
        ;; If you choose to terminate special buffers by removing the following,
        ;; ensure that the special buffers you want to keep are added
        ;; keep-buffer-name rules above.
        ;;
        ;; DO NOT REMOVE special buffers unless you are certain of what
        ;; you are doing.
        (keep-buffer-type . "special")

        ;; Retain visible buffers are those currently displayed in any window.
        ;; Keep this in the end, after all the rules above.
        ;; It is generally discouraged to set this to nil, as doing so may result
        ;; in the termination of visible buffers, except for the currently active
        ;; buffer in the selected window.
        ;;
        ;; DO NOT REMOVE visible buffers unless necessary.
        (keep-buffer-status . "visible")

        ;; Kill the remaining buffers that were not retained by previous rules
        (return . :kill)))
```

### Predicate function

You can set a custom predicate function using `buffer-terminator-predicate` to control which inactive buffers the *buffer-terminator* should keep or kill based on specific conditions.

Here is an example of how to define a custom predicate function:

``` elisp
(defun my-buffer-terminator-predicate ()
  "Function to decide the fate of a buffer.
:kill    Indicates that the buffer should be killed.
:keep    Indicates that the buffer should be kept.
nil      Let Buffer-Terminator decide."
  (let* ((buffer (current-buffer))
         (buffer-name (buffer-name buffer)))
    (cond
     ;; Kill the scratch buffer
     ((string= buffer-name "*scratch*")
      :kill)

     ;; Keep this buffer:
     ((string= buffer-name "my-precious-buffer")
      :keep)

     (t
      ;; Nil = Buffer-Terminator decides
      nil))))

(setq buffer-terminator-predicate #'my-buffer-terminator-predicate)
```

## Frequently asked questions

### Why? What problem is this aiming to solve?

- Some users prefer terminating inactive buffers to improve Emacs' performance by reducing the number of open buffers. This, in turn, decreases the load from active modes, timers, and other processes associated with those buffers. Buffer-local modes and their timers consume both CPU and memory. Why keep them alive when they can be safely removed?
- Some users prefer to keep only the buffers they actively need open, helping to declutter the buffer list. Decluttering the buffer list can also improve the performance of other packages. For example, saving and loading an [easysession](https://github.com/jamescherti/easysession.el) or desktop.el is much faster when the buffer list is reduced.
- Some users prefer that buffers not part of an active window be automatically closed, as they are not actively needed.
- Some Emacs packages continue interacting with open buffers, even when they are buried ([Reddit post: A function to periodically wipe buffers not recently shown; thoughts?](https://www.reddit.com/r/emacs/comments/1h15mni/a_function_to_periodically_wipe_buffers_not/)).

### How is this different from the builtin midnight-mode?

Midnight-mode does not address the problem that buffer-terminator solves, which is the safe and frequent termination of inactive buffers.

Midnight mode and `clean-buffer-list` are for killing buffers once a day. The Midnight option `clean-buffer-list-delay-general` specifies the number of days before a buffer becomes eligible for auto-killing, rather than using seconds or minutes as a timeout.

In contrast, `buffer-terminator` allows specifying the timeout interval in seconds (Default: 30 minutes), enabling more frequent termination of inactive buffers.

The `buffer-terminator` package offers additional features that are not supported by midnight and `clean-buffer-list`, including:
- Buffer-terminator does not kill visible buffers in other tabs, even if they exceed the timeout. This prevents disruptions to editing workflows.
Buffer-terminator provides the option to choose whether to keep or kill specific types of buffers, such as those associated with processes or file-visiting buffers.
- Buffer-terminator avoids relying on `buffer-display-time`, which is not always updated reliably. For instance, `buffer-display-time` may not reflect activity when switching to a window or tab displaying a specific buffer.
- Buffer-terminator does not kill special buffers by default, whereas Midnight kills all special buffers by default unless the user tells Midnight to ignore them. Midnight's behavior can disrupt packages like Corfu, Cape, Consult, Eglot, Flymake, and others that rely on special buffers to store data.
- Buffer-terminator can also kill specific special buffers. It is useful, for example, if the user want to keep special buffers (i.e. using buffer-terminator-keep-special-buffers), but with a few exceptions: The user still want to kill *Help* and *helpful ...* buffers (and maybe some other buffers related to documentation) if they weren't used for a while.

## Author and License

The *buffer-terminator* Emacs package has been written by [James Cherti](https://www.jamescherti.com/) and is distributed under terms of the GNU General Public License version 3, or, at your choice, any later version.

Copyright (C) 2024 James Cherti

This program is free software: you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation, either version 3 of the License, or (at your option) any later version. This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details. You should have received a copy of the GNU General Public License along with this program.

## Links

- [buffer-terminator.el @GitHub](https://github.com/jamescherti/buffer-terminator.el)

Other Emacs packages by the same author:
- [minimal-emacs.d](https://github.com/jamescherti/minimal-emacs.d): This repository hosts a minimal Emacs configuration designed to serve as a foundation for your vanilla Emacs setup and provide a solid base for an enhanced Emacs experience.
- [compile-angel.el](https://github.com/jamescherti/compile-angel.el): **Speed up Emacs!** This package guarantees that all .el files are both byte-compiled and native-compiled, which significantly speeds up Emacs.
- [outline-indent.el](https://github.com/jamescherti/outline-indent.el): An Emacs package that provides a minor mode that enables code folding and outlining based on indentation levels for various indentation-based text files, such as YAML, Python, and other indented text files.
- [easysession.el](https://github.com/jamescherti/easysession.el): Easysession is lightweight Emacs session manager that can persist and restore file editing buffers, indirect buffers/clones, Dired buffers, the tab-bar, and the Emacs frames (with or without the Emacs frames size, width, and height).
- [vim-tab-bar.el](https://github.com/jamescherti/vim-tab-bar.el): Make the Emacs tab-bar Look Like Vim’s Tab Bar.
- [elispcomp](https://github.com/jamescherti/elispcomp): A command line tool that allows compiling Elisp code directly from the terminal or from a shell script. It facilitates the generation of optimized .elc (byte-compiled) and .eln (native-compiled) files.
- [tomorrow-night-deepblue-theme.el](https://github.com/jamescherti/tomorrow-night-deepblue-theme.el): The Tomorrow Night Deepblue Emacs theme is a beautiful deep blue variant of the Tomorrow Night theme, which is renowned for its elegant color palette that is pleasing to the eyes. It features a deep blue background color that creates a calming atmosphere. The theme is also a great choice for those who miss the blue themes that were trendy a few years ago.
- [Ultyas](https://github.com/jamescherti/ultyas/): A command-line tool designed to simplify the process of converting code snippets from UltiSnips to YASnippet format.
- [dir-config.el](https://github.com/jamescherti/dir-config.el): Automatically find and evaluate .dir-config.el Elisp files to configure directory-specific settings.
- [flymake-bashate.el](https://github.com/jamescherti/flymake-bashate.el): A package that provides a Flymake backend for the bashate Bash script style checker.
- [flymake-ansible-lint.el](https://github.com/jamescherti/flymake-ansible-lint.el): An Emacs package that offers a Flymake backend for ansible-lint.
- [inhibit-mouse.el](https://github.com/jamescherti/inhibit-mouse.el): A package that disables mouse input in Emacs, offering a simpler and faster alternative to the disable-mouse package.
- [quick-sdcv.el](https://github.com/jamescherti/quick-sdcv.el): This package enables Emacs to function as an offline dictionary by using the sdcv command-line tool directly within Emacs.
- [enhanced-evil-paredit.el](https://github.com/jamescherti/enhanced-evil-paredit.el): An Emacs package that prevents parenthesis imbalance when using *evil-mode* with *paredit*. It intercepts *evil-mode* commands such as delete, change, and paste, blocking their execution if they would break the parenthetical structure.
