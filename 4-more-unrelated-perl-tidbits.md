# 4 More Unrelated Perl Tidbits

Last year I wrote an article titled [4 Unrelated Perl Tidbits](https://dev.to/nicholasbhubbard/4-unrelated-perl-tidbits-2766), where I talked about some random Perl facts I learned about from reading [Programming Perl](https://www.oreilly.com/library/view/programming-perl-4th/9781449321451/). In this article I will talk about 4 more random and interesting Perl features I have learned about since.

### Built-Ins Can Be Overridden with Lexical Subroutines

Perl version 5.18 introduced [lexical subroutines](https://perldoc.perl.org/perlsub#Lexical-Subroutines), which are often referred to as "my subs". An interesting characteristic of lexical subs is that unlike regular subroutines, they can override built-ins.

```perl
use v5.18;

my sub print {
    die "printing is banned\n";
}

print "Hello, World!\n";

__END__

$ perl tmp.pl
printing is banned
```

If anybody has seen a legitimate use of this feature then please comment below.

### Goto Searches For Labels In The Dynamic Scope

[Goto](https://perldoc.perl.org/functions/goto) searches for labels from within its [dynamic scope](https://en.wikipedia.org/wiki/Scope_(computer_science)#Dynamic_scope).

```perl
sub foo {
    bar();
}

sub bar {
    goto LABEL;
}

sub baz {
  LABEL:
    print "hello from after LABEL\n";
    foo();
}

baz();

__END__

$ perl tmp.pl
hello from after LABEL
hello from after LABEL
hello from after LABEL
...
```

This program just goes on forever printing `hello from after LABEL`.

### Recursive Anonymous Subroutines With __SUB__

Perl version 5.16 introduced the [__SUB__](https://perldoc.perl.org/functions/__SUB__) special token that holds a reference to the current subroutine. You can use `__SUB__` to make a recursive call in an anonymous subroutine.

```perl
use v5.16;

higher_order_function(
    sub {
        my $n = shift;
        if ($n <= 1) { return 1 }
        else         { return $n * __SUB__->($n-1) }
    }
)
```

### Regex Modifier For Only Portions Of The Regex

You can use the `(?M:)` pattern in a regex to turn on the modifier specified by `M`, only inside the parentheses. For example, the following two regexs are the same:

```perl
/foo/i
/(?i:foo)/
```

You can also turn a modifier off with `(?-M:)`, which is shown in this example:

```perl
if ('FOO' =~ /(?-i:foo)/i) {
    print "matches\n"
} else {
    print "does not match\n"
}

__END__

$ perl tmp.pl
does not match
```

This feature is useful if you want to turn a modifier on/off for only a portion of the regex:

```perl
if ('fooBAR' =~ /(?-i:foo)bar/i) {
    print "matches\n"
} else {
    print "does not match\n"
}

__END__

$ perl tmp.pl
matches
```
