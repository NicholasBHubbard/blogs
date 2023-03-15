# An Interesting Perl Pattern That Doesn't Work

I recently came up with an interesting pattern that is supposed to use a closure to return a read-only configuration. Unfortunately, this pattern has a terrible flaw.

```perl
{
    my %config;

    sub config {
        return %config if %config;
        %config = create_config();
        return %config;
    }
}
```

The `config()` subroutine is a lexical closure over the `%config` hash. No other code in a program would be able to access the `%config` variable, as everything is defined in its own [block](https://perldoc.perl.org/perlsyn#Basic-BLOCKs).

As an aside, this pattern could just as easily be written with a [state variable](https://perldoc.perl.org/functions/state), but I find it harder to explain the pattern when done this way.

If you already understands blocks and closures, then you may want to skip their explanations and go straight to an explanation of the actual problem [here](# The Problem).

# Blocks

To understand this pattern we first have to understand how blocks work. Here is a code example that shows how blocks work:

```perl
{
    my $var = 'foo';
    print "from inside the block where \$var is defined \$var = $var\n";

    {
        print "from the most inner block \$var = $var\n";
    }
}

print "from outside the block \$var = $var\n";

__END__

$ perl tmp.pl
from inside the block where $var is defined $var = foo
from the most inner block $var = foo
from outside the block $var = 
```

This code example shows that a block introduces a lexical scope. Variables defined in a lexical scope are only available in its defining scope, and lexical scopes nested inside their defining scope. We can see that the `$var` variable is available in the scope it is defined in, and from the scope nested in its defining scope. However, outside of its defining scope, `$var` is not defined.

If we turn on `strict` we get a fatal compilation error for trying to use `$var` from outside its defining scope:

```
Global symbol "$var" requires explicit package name (did you forget to declare "my $var"?) at tmp.pl line 14.
Execution of tmp.pl aborted due to compilation errors.
```

You may be wondering if the `config()` subroutine is accessible from outside the block it was defined in. The answer that it is indeed available, because subroutine declarations are always global to the current package. This code example shows this fact:

```perl
use strict;

{
    sub foo {
        print "hello from foo!\n";
    }
}

foo();

__END__

$ perl tmp.pl
hello from foo!
```

# Closures

Now that we understand blocks we can understand closures. In Perl, a closure is a subroutine that has access to the lexical environment that it was defined in. Here is the classic example:

```perl
use strict;

{
    my $n = 0;        

    sub increment {
        $n += 1;
        return $n;
    }

    sub decrement {
        $n -= 1;
        return $n;
    }
}

print 'increment() -> ', increment(), "\n";
print 'increment() -> ', increment(), "\n";
print 'decrement() -> ', decrement(), "\n";
print 'increment() -> ', increment(), "\n";
print 'increment() -> ', increment(), "\n";
print 'decrement() -> ', decrement(), "\n";

__END__

$ perl tmp.pl
increment() -> 1
increment() -> 2
decrement() -> 1
increment() -> 2
increment() -> 3
decrement() -> 2
```

The `increment()` and `decrement()` subroutines are both able to access the `$n` variable, though no other subroutines in a larger program would be able to, due to the outer block. For this reason the `increment()` and `decrement()` subroutines are closures over `$n`;

# The Problem

We should now have all the knowledge needed to understand the pattern that this article is about.

The idea of the pattern is that if the `%config` variable has already been set then we just return it, and otherwise we set its value before returning it. This means that `%config` will only be set the first time that we call `config()`, and on all subsequent calls it will simply be returned. Therefore `config()` can be thought of as a function constant ... right?

Here is a code example where our pattern works as expected:

```perl
use strict;

{
    my %config;

    sub config {
        return %config if %config;
        %config = create_config();
        return %config;
    }
}

sub create_config {
    print "hello from create_config()\n";
    return (foo => 12);
}

my %config1 = config();

$config1{foo} = 1004;

my %config2 = config();

print "%config2's foo key = $config2{foo}\n";

__END__

$ perl tmp.pl
hello from create_config()
%config2's foo key = 12
```

This output displays a couple of important points. First, we know that the `create_config()` subroutine was only invoked a single time, even though we invoked `config()` twice. We know this because the "hello from create_config()" message is only printed a single time. The other important thing to note is that because we got the output "%config2's foo key = 12", we know that our modification of `%config1`'s `foo` key (which we set to 1004), did not effect the `%config` variable that our `config()` subroutine closes over. If it had, then `%config2`'s `foo` key would associate to `1004`.

So what is the problem? Well ... everything falls apart when the `%config` variable is set to a multi-dimensional data structure. The following code encapsulates the fundamental problem with our pattern:

```perl
use strict;

{
    my %config;

    sub config {
        return %config if %config;
        %config = create_config();
        return %config;
    }
}

sub create_config {
    return (foo => [1, 2, 3]);
}

my %config1 = config();

$config1{foo}->[0] = 1004;

my %config2 = config();

print "%config2's foo key = [", join(', ', @{$config2{foo}}), "]\n";

__END__

$ perl tmp.pl
%config2's foo key = [1004, 2, 3]
```

Uh oh! We were able to mutate the `%config` variable that `config()` closes over, which means that `config()` is not actually a constant function. Now we come to the fundamental problem of our pattern. Because in Perl multi-dimensional data structures are made up of references, and perl does not perform [deep-copying](https://en.wikipedia.org/wiki/Object_copying#Deep_copy) by default, we are able to mutate the underlying references of multi-dimensional data structures.

Here is a code example that shows that Perl does not perform deep-copying:

```perl
use strict;

my @array1 = ([1, 2, 3], [4, 5, 6], [7, 8, 9]);

print '@array1 contents:', "\n";
for my $elem (@array1) {
    print "    $elem\n";
}

# copy @array1 to @array2
my @array2 = @array1;

print '@array2 contents:', "\n";
for my $elem (@array2) {
       print "    $elem\n";
}

__END__

$ perl tmp.pl
@array1 contents:
    ARRAY(0x13875e8)
    ARRAY(0x13d1ef8)
    ARRAY(0x13d1fe8)
@array2 contents:
    ARRAY(0x13875e8)
    ARRAY(0x13d1ef8)
    ARRAY(0x13d1fe8)
```

In this programs output we can see that `@array1` and `@array2` contain the exact same references, which means that Perl does not perform deep-copying. If Perl did perform deep-copying, then when we copied `@array1` into `@array2`, Perl would have made (recursive) copies of all the references in `@array1` into new refererences. Perl's lack of deep-copying is the fundamental flaw of our pattern, as it means that we can modify `%config`'s references from its copies that are returned by `config()`.

# Solutions

There are many ways we can solve this problem. First, we could use [lock_hash_recurse](https://perldoc.perl.org/Hash::Util#lock_hash_recurse) from the core [Hash::Util](https://perldoc.perl.org/Hash::Util) module to lock `%config`. After locking `%config`, we would get an error if we tried to mutate any of its values.

We could also use [Const::Fast](https://metacpan.org/pod/Const::Fast) from CPAN to make `%config` an actual read-only hash. Similarly to locking the hash, we would get an error if we tried to mutate `%config`.

Finally, we could use [Clone](https://metacpan.org/pod/Clone) from CPAN to return a deep-copy of `%config` from the `config()` subroutine. Unlike the other solutions, our code could freely modify copies of `%config` without getting any errors, but these modifications would not effect the actual `%config`.
