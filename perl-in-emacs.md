This post is about how I use Emacs to write Perl. I do not claim to have the best Perl setup of all time or anything like that. The features I need to write Perl effectively are syntax highlighting, auto-indentation, linting, and code navigation.

I personally like to build my own IDE by bringing together unrelated packages, which is in contrast to full blown IDE packages, such as [Devel::PerlySense](https://metacpan.org/pod/Devel::PerlySense) or [Perl::LanguageServer](https://metacpan.org/pod/Perl::LanguageServer). These packages just aren't for me.


<a id="orga25e1f5"></a>

# Basics

By default Emacs uses perl-mode instead of the more advanced cperl-mode. Both packages are built-in, so to use cperl-mode instead of perl-mode all you have to do is add the following line to your config.

```lisp
(fset 'perl-mode 'cperl-mode)
```

The cperl-mode that was released with Emacs 28 improved the syntax highlighting for regular expressions and heredocs. It also fixed an annoying bug where array variable names in comments were highlighted with the array face instead of the comment face.

If you are using an Emacs version less than 28 then I would recommend downloading the [cperl-mode off the Emacs 28 branch](https://github.com/emacs-mirror/emacs/blob/emacs-28/lisp/progmodes/cperl-mode.el). I personally place this file in `~/.emacs.d/cperl-mode/cperl-mode.el`, then I load it with the following code.

```lisp
(add-to-list 'load "~/.emacs.d/cperl-mode")
(require 'cperl-mode)
```

By default cperl-mode replaces trailing whitespace with underscores. I cannot imagine why you would want this. To turn it off add the following line to your config.

```lisp
(setq cperl-invalid-face nil)
```

cperl-mode indents code by 2 spaces by default. You can modify this by setting the `cperl-indent-level` variable.

You probably want multi-line statements wrapped in parens to be indented like a block. For example by default cperl-mode indents this hash declaration in a strange way.

```perl
my %hash = (
            'foo' => 1,
            'bar' => 2,
            'baz' => 3
           );
```
To fix this add the following to your config.

```lisp
(setq cperl-indent-parens-as-block t)
(setq cperl-close-paren-offset (- cperl-indent-level))
```

Now our hash declaration indents nicely!

```perl
my %hash = (
    'foo' => 1,
    'bar' => 2,
    'baz' => 3
);
```

<a id="org3c40091"></a>

# Linting

Linting our Perl code helps us easily find bugs caused by typos. My favorite Emacs linting package is [Flycheck](https://www.flycheck.org/en/latest/), which comes with built-in support for Perl.

By default Flycheck checks your code with the Perl interpreter, but it also comes with integration with [Perl::Critic](https://metacpan.org/pod/Perl::Critic). Personally I have only used the former.

I like to lint the file everytime I save, and I like to display any errors immediately. Here is how I accomplish this with Flycheck.

```lisp
(require 'flycheck)
(setq flycheck-check-syntax-automatically '(mode-enabled save))
(setq flycheck-display-errors-delay 0.1)
```

To enable flycheck mode in cperl-mode, simply turn it on with a hook.

```
(add-hook 'cperl-mode-hook 'flycheck-mode)
```

Now Emacs will underline any syntax errors, and you can view the message in the echo area by placing your cursor on the erroneus code.

I cannot tell you how many simple errors you will catch just by using Flycheck!


<a id="org063479d"></a>

# Code Navigation

For jumping between function definitions I use [dumb-jump](https://github.com/jacktasia/dumb-jump), which usually **just works**. I configure dumb-jump to use [ag](https://github.com/ggreer/the_silver_searcher) for its searching which makes it work very quickly.

```lisp
(require 'dumb-jump)
(setq dumb-jump-force-searcher 'ag)
(add-hook 'xref-backend-functions #'dumb-jump-xref-activate)
```

I can then use dumb-jump by calling the `xref-find-definitions` function as my cursor is on the symbol I want to search for. This function is bound to `M-.` by default.


<a id="orgde68f03"></a>

# Shell

A lot of people use `M-x compile` to run their code, and one of the various debugger packages to run the Perl debugger. Personally I just use plain old [Bash](https://www.gnu.org/software/bash/) with the built-in `M-x shell`. This makes my work flow when it comes to running and debugging quite similar to that of a classic Perl vimmer who does all their work in a terminal.

I use the wonderful [shx](https://github.com/riscy/shx-for-emacs) package for making `M-x shell` a more usable shell, and I use [shell-pop](https://github.com/kyagi/shell-pop-el) for popping up shell buffers that are automatically cd'd to the current files directory.

```
(require 'shx)
(add-hook 'shell-mode-hook 'shx-mode)

(require 'shell-pop)
(setq shell-pop-autocd-to-working-dir t)
(global-set-key (kbd "M-SPC") 'shell-pop)
```

<a id="org3936e96"></a>

# Closing Thoughts

Every 3rd-party package I described in this post is useful not only for Perl, but for programming in any language. This gives a uniform experience across different programming languages. If I instead used one of the Perl IDE packages then I wouldn't get the same uniform experience when using other languages.


<a id="orgd51f8f0"></a>

# See Also

-   [CPerl Documentation](https://www.emacswiki.org/emacs/CPerlMode)  - Offical documentation for cperl-mode
-   [Perl::LanguageServer](https://metacpan.org/pod/Perl::LanguageServer) - Language server for Perl
-   [Devel::PerlySense](https://metacpan.org/pod/Devel::PerlySense)    - Perl IDE features for Emacs
-   [Emacs::PDE](https://metacpan.org/pod/Emacs::PDE)           - Elisp extensions for Perl development

