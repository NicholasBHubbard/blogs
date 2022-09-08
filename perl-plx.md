[App::plx](https://metacpan.org/pod/App::plx) (Plx) is a tool for configuring per-project Perl development environments.

Plx is not difficult to use and has very clear documentation. In this post I will give a brief overview of some of the problems that it solves and how to get started using it.

This is not a complete overview on everything that Plx can do.


<a id="orgbd602ea"></a>

# Why Plx?

Imagine you have a project that depends on a Perl version greater than that of your systems built-in Perl. You will need to install the correct version of Perl and figure out how to get your project to use this Perl instead of the system's Perl.

Next imagine a scenario where you are working on two different Perl projects that depend on a different version of the same CPAN module. You will need to figure out how to install both versions into different locations, and then you will need to figure out how to locate the correct version from the two different projects.

These are two important problems that Plx solves.


<a id="org87e7c16"></a>

# Installation

Before you can use Plx you must install Plx.

If you already have a CPAN installer, such as [cpanminus](https://metacpan.org/pod/App::cpanminus), then you should probably just use that to install Plx.

Plx can also be bootstrapped into a self contained script like so:

    $ mkdir "$HOME/bin"
    $ wget https://raw.githubusercontent.com/shadowcat-mst/plx/master/bin/plx-packed -O "$HOME/bin/plx"
    $ chmod +x "$HOME/bin/plx"

You can install Plx into any directory, I just chose "$HOME/bin" for simplicity. Just make sure you pick a directory in your [PATH](https://en.wikipedia.org/wiki/PATH_(variable)).


<a id="orgd968283"></a>

# Initialization

If you want to use Plx for your Perl project, you must first initialize the project to use Plx. To do this we must `cd` into the root directory of the project and then execute Plx with the `--init` flag.

The `--init` flag behaves differently depending on its argument.

When `--init` is called with a file path, it assumes it is the path is to a Perl interpreter and sets up Plx to use it.

    $ plx --init /path/to/some/perl

When called with `perl` as the argument it sets up Plx to use the first Perl in your [PATH](https://en.wikipedia.org/wiki/PATH_(variable)).

    $ plx --init perl

The final and most exciting way to call `--init` is with a Perl version number. When called with a version number, Plx will look for a Perl of the given version first in your [PATH](https://en.wikipedia.org/wiki/PATH_(variable)) and otherwise via [Perlbrew](https://perlbrew.pl/).

    $ plx --init 5.36.0

After initializing Plx you can execute your project code with a command like `$ plx /path/to/project/script.pl`, and Plx will execute the script with the Perl interpreter you specified with the `plx --init` command.

This is how Plx solves the problem of needing to use a different Perl than your systems built-in Perl.


<a id="org124c9de"></a>

# Installing CPAN Modules

My favorite feature of Plx is that it allows you to install modules off of CPAN into a [local::lib](https://metacpan.org/pod/local::lib) using [cpanminus](https://metacpan.org/pod/App::cpanminus). This allows you to segregate your CPAN modules dependencies on a per-project basis.

To do this we must `cd` into the root directory of our project and run the following command.

    $ plx --cpanm -Llocal Some::Module

This will install `Some::Module` into a project-local library located in a directory named `local/lib` at the root of the project.

This solves the problem of two projects requiring different versions of the same CPAN module. If both projects use Plx they can simply install their desired version into a [local::lib](https://metacpan.org/pod/local::lib).


<a id="org9b2111c"></a>

# Userstrap

What if you want to use your own Perl interpreter and a [local::lib](https://metacpan.org/pod/local::lib) when you are working outside of a dedicated Plx project?

Plx has a `--userstrap` flag that will set this up for you automatically.

    $ plx --userstrap /path/to/some/perl

Calling `--userstrap` essentially sets up your `$HOME` to be a Plx project and sets up a [local::lib](https://metacpan.org/pod/local::lib) in `$HOME/perl5`, installs [App::plx](https://metacpan.org/pod/App::plx) and [App::cpanminus](https://metacpan.org/pod/App::cpanminus) into the local::lib, and adds a line to your `$HOME/.bashrc` that sets up Plx for your Bash shell.

Now when you run Plx from outside a dedicated Plx project it will use `$HOME` as a sort of default Plx project. You can use `--userstrap` to prevent needing to use your system Perl, so you and can instead always use Plx.

Note that `--userstrap` requires that you use a Bash shell.


<a id="org285989d"></a>

# Plx is For Everybody

Plx is designed to not only provide a nice experience for Perl developers, but also to be usable by a sysadmin that isn't a Perl expert. Therefore Plx is configured through simple text files that can be manipulated by hand, and allows multiple commands to be run in a single Plx invocation via the `--multi` flag, which makes scripting Plx cleaner.


<a id="orgdee387b"></a>

# Synopsis

Plx is a tool for creating per-project virtual Perl environments. Plx lets us avoid a lot of headaches that come with developing multiple Perl projects on the same system.

A lot of what Plx does can be done by combining features of other CPAN modules, but Plx brings together these functionalities in a way that is easy to use and understand.

This blog post is only a brief introduction to Plx. Please go on to read the manual for more a more detailed overview of its features.


<a id="orgef3ab18"></a>

# Bonus Tip for Emacs Users

If you lint your Perl code with the Perl interpreter using Flycheck, you will need to determine if the buffer is part of a Plx project so it runs the Perl interpreter through Plx.

Use the following code to do this:

```lisp
(require 'flycheck)
(require 'projectile)

(add-hook 'cperl-mode-hook 'flycheck-mode)
(add-hook 'cperl-mode-hook 'my/cperl-select-correct-flycheck-checker)

(flycheck-define-checker my/perl-plx
  :command ("plx" "-w" "-c"
            (option-list "-I" flycheck-perl-include-path)
            (option-list "-M" flycheck-perl-module-list concat))
  :standard-input t
  :error-patterns
  ((error line-start (minimal-match (message))
          " at - line " line
          (or "." (and ", " (zero-or-more not-newline))) line-end))
  :modes (perl-mode cperl-mode))

(defun my/cperl-select-correct-flycheck-checker ()
  "If the current buffer is part of a plx project then use the `my/perl-plx'
checker, otherwise use the `perl' checker."
  (let ((proj-root (projectile-project-root)))
    (if (and proj-root (file-directory-p (concat proj-root ".plx")))
        (flycheck-select-checker 'my/perl-plx)
      (flycheck-select-checker 'perl))))
```