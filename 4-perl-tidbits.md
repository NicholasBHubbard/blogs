I recently acquired a copy of [Programming Perl](https://www.oreilly.com/library/view/programming-perl-4th/9781449321451/), often referred to as "The Camel Book" in the Perl community. After reading the first 4 chapters I thought I would share a few Perl tidbits I found interesting.


<a id="org1d47756"></a>

# While <> is True

Say we have the following file named `logos.txt`.

    Onion
    Camel
    
    Raptor

And in the same directory we have the following program named `scratch.pl`.

    open my $fh, '<', 'logos.txt' or die;
    while (my $logo = <$fh>) {
        print $logo;
    }
    close $fh;

As expected our program prints the contents of `logos.txt`.

    $ perl ./scratch.pl
    Onion
    Camel
    
    Raptor

Something I never really considered though, is why doesn't the loop exit when `<>` reads the empty line in `logos.txt`? Shouldn't `<>` return an empty string which is a false value?

The reason why the loop doesn't exit is because `<>` reads the newline at the end of the line so we actually get `"\n"` which is a true value.


<a id="org99a62c3"></a>

# Heredocs Can Execute Shell Commands

Most Perl programmers know that if you single quote a heredocs terminating string you prevent variable interpolation.

    my $var = 12;
    print <<'EOS';
    Hello
    $var = 12
    EOS

You can see in the programs output that `$var` was not interpolated.

    $ perl ./scratch.pl
    Hello
    $var = 12

But did you know that if you backquote the terminating string then each line is executed as a shell command?

    print <<`EOC`;
    echo this is a shell command
    echo this is also a shell command
    EOC

When we run this program we can see that the echo commands were executed.

    $ perl ./scratch.pl
    this is a shell command
    this is also a shell command


<a id="orgd568d68"></a>

# The Comma Operator

I always took commas for granted, never realizing they were actually an operator.

Did you ever wonder why lists return their last element when evaluated in scalar context? Turns out it is due to the comma operator.

In scalar context the comma operator "," ignores its first argument, and returns its second element evaluated in scalar context.

This means that in scalar context the list `(11, 22, 33)` will evaluate to 33. The first comma operator will throw away the 11 and then return `(22, 33)` evaluated in scalar context, which will be evaluated by throwing away the 22 and returning 33.


<a id="org644e91b"></a>

# Auto-Incrementing Strings

In perl you can not only auto-increment numbers, but also strings. To increment a string it must match the regex `/^[a-zA-Z]*[0-9]*\z/`.

Single alphabet character strings are incremented intuitively.

    $ perl -E 'my $v = "a"; say ++$v'
    b
    $ perl -E 'my $v1 = "B"; say ++$v1'
    C

What happens though if we increment a non-alphanumeric char. Will it give the next ASCII character? Turns out non-alphanumeric characters are treated as 0's when incrementing. Fortunately if we use [warnings](https://perldoc.perl.org/warnings) Perl will give us a heads up.

    $ perl -W -E 'my $v = "-"; say ++$v'
    Argument "-" treated as 0 in increment (++) at -e line 1.
    1

What happens if we increment `z`, which is at the end of the alphabet? Do we wrap back around to `a`? Turns out this is where things get interesting. Perl increments the string just like you would in a regular number system. Just like `9 + 1 = 10` in decimal `z + 1 = aa` in &#x2026; stringimal.

    $ perl -E 'my $v = "z"; say ++$v'
    aa

Here are some more examples to show you the magic of string auto-incrementing.

    $ perl -W -E 'my $v1 = "foo"; say ++$v1'
    fop
    $ perl -W -E 'my $v1 = "az"; say ++$v1'
    ba
    $ perl -W -E 'my $v1 = "a9"; say ++$v1'
    b0