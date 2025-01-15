How I used a named pipe to save memory (in Perl)

I recently ran into an interesting bug in my Slackware package manager [sbozyp](https://github.com/NicholasBHubbard/sbozyp). In this post I will detail the bug and my solution.

Disclaimer: the code snippets in this post do not deal with every edge case and are only meant to convey the ideas relevant to this blog post.

Sbozyp is a package manager for [SlackBuilds.org](https://www.slackbuilds.org/), a repository of build scripts for Slackware packages. There is a very important subroutine in sbozyp named `build_slackware_pkg()` that (basically) takes the name of a package, builds it into a Slackware package, and returns the path to the built package. To build the package we must execute the packages SlackBuild script. A SlackBuild script always has the general form: { basic setup -> perform build procedure -> conglomerate built parts into a Slackware package }. The last step is always performed by a Slackware-specific tool called [makepkg](http://www.slackware.com/config/packages.php). There is no way (that I can think of) to be able to determine the name of the Slackware package before executing the SlackBuild script. For this reason sbozyp must inspect the stdout of the SlackBuild script to look for a line that makepkg always outputs that looks like `Slackware package $PATH created.`. It is also important of course that the user can see the output of the SlackBuild script as it may contain vital information. This means that both my program and the user needs to examine the output of the SlackBuild script. (See [here](https://www.slackwiki.com/SlackBuild_Scripts) for more information on SlackBuild scripts )

With the problem description out of the way we can now focus on the solution.

My original solution was an obvious one but had an interesting bug. I simply would use Perl's [system](https://perldoc.perl.org/functions/system) command to execute the SlackBuild script from a shell, where I would pipe the stdout to [tee](https://en.wikipedia.org/wiki/Tee_(command)), writing the stdout to a temporary file. After the SlackBuild would execute I would read the temporary file looking the "Slackware package $PATH created." line. This solved the problem of allowing both the user and sbozyp to read the stdout of the SlackBuild script. For 99% of builds this worked fine ... but then there was [qemu](https://www.qemu.org/). Qemu is a massive project that requires a huge compilation. My system uses a 4GB tmpfs mounted at /tmp, and the temporary file that was being teed to ended up getting so large from all the qemu compilation output that tee crashed for "device out of space".

Here is a basic (not actually realistic to sbozyp) outline of that code:

```perl
sub build_slackware_pkg {
    my ($pkg_name) = @_;
    my $slackbuild_script = find_slackbuild_script($pkg_name);
    my $tmp_file = make_temp_file(DIR=>'/tmp');
    0 == system("set -o pipefail; $slackbuild_script | tee $tmp_file") or die;
    open my $fh, '<', $tmp_file or die;
    my $slackware_pkg;
    while (my $stdout_line = <$tmp_file>) {
        $slackware_pkg = $1 if $stdout_line =~ /^Slackware package (.+) created\.$/;
    }
    return $slackware_pkg;
}
```

To solve the problem I create a [named pipe](https://en.wikipedia.org/wiki/Named_pipe), fork, and use the child process to execute the SlackBuild script, teeing stdout to the named pipe. From the parent I read the named pipe looking for the magic "Slackware package $PATH created." line. Here is a (also not realistic to sbozyp) outline of this code:

```perl
use POSIX qw(mkfifo WNOHANG);

sub build_slackware_pkg {
    my ($pkg_name) = @_;
    my $slackbuild_script = find_slackbuild_script($pkg_name);
    my $tmp_dir = make_temp_dir(DIR=>'/tmp');
    mkfifo("$tmp_dir/fifo", 0700);
    my $pid = fork();
    if ($pid == 0) { # child
        0 == system("set -o pipefail; $slackbuild_script | tee $fifo") or die;
        exit 0;
    } else { # parent
        open my $fifo_r_fh, '<', $fifo or die;
        my $slackware_pkg;
        while (waitpid($pid, WNOHANG) == 0 or my $stdout_line = <$fifo_r_fh>) {
            $slackware_pkg = $1 if $stdout_line and $stdout_line =~ /^Slackware package (.+) created\.$/;
        }
        die if $? != 0; # $? has exit status of child
        return $slackware_pkg;
    }
}
```

This implementation saves disk space as the data is being teed to a named pipe instead of a regular file in /tmp. As data is written to the pipe it is immediately being read, throwing away all the data that sbozyp doesn't need.
