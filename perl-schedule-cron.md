A [daemon](https://en.wikipedia.org/wiki/Daemon_(computing)) is a program that runs in the background for an indefinite period of time.

An important daemon on Unix like operating systems is the [cron-scheduler](https://en.wikipedia.org/wiki/Cron) that can be configured to perform tasks periodically. Though there are many different types of daemons we will explore how to create a cron daemon.

My favorite CPAN module for creating a daemon is [Schedule::Cron](https://metacpan.org/pod/Schedule::Cron), which allows us to create a daemon that dispatches Perl subroutines at preconfigured intervals. The reason I like [Schedule::Cron](https://metacpan.org/pod/Schedule::Cron) is that it is easy to learn, easy to understand, and it works.

This article is not a complete overview of everything that [Schedule::Cron](https://metacpan.org/pod/Schedule::Cron) can do so be sure to go on to read the official documentation afterwards.


<a id="orga259ded"></a>

# A Simple Daemon

Lets write a simple (and useless) daemon that logs the current time every 5 minutes, and rotates the log file every hour, keeping at most 4 old logs. The current logfile will be named `$HOME/times.txt`, and the old log files will be named `$HOME/times.txt.{1,2,3,4}`.

First we need a function to write the time to the current log file.

sub append_time {
    my $logfile = "$ENV{HOME}/times.txt";
    open my $fh, '>>', $logfile or die "cannot open file '$logfile': $!";
    my $time = localtime();
    print $fh "$time\n";
    close $fh;
}

Next we need a function to rotate the old log files.

    use File::Copy 'move';
    
    sub rotate_time_log {
        my $logfile      = "$ENV{HOME}/times.txt";
        my @old_logfiles = grep /^$ENV{HOME}\/times\.txt\.\d+$/, glob "$ENV{HOME}/*";
        # We don't need to rotate unless we have more than 4 old logfiles
        return if @old_logfiles <= 4;
        unlink "$logfile.4";
        for (my $i = 3; $i >= 1; $i--) {
            move $old_logfiles[$i], $logfile.$i+1;
        }
        move $logfile, "$logfile.1";
    }

Understanding how these functions work is not important for learning about [Schedule::Cron](https://metacpan.org/pod/Schedule::Cron).

Now that we have our time logging functions lets initialize our daemon object. To initialize the daemon object we will use the [Schedule::Cron::new](https://metacpan.org/pod/Schedule::Cron#$cron-=-new-Schedule::Cron($dispatcher,[extra-args])) function.

The first argument to [Schedule::Cron::new](https://metacpan.org/pod/Schedule::Cron#$cron-=-new-Schedule::Cron($dispatcher,[extra-args])) must be a reference to a subroutine that will be used as a default if we add a cron entry without specifying the function we want to run. This is only useful if there is only one function we want our daemon to run. We won't be using this feature so we just set it to a function that kills the program.

There are many options we can pass to [Schedule::Cron::new](https://metacpan.org/pod/Schedule::Cron#$cron-=-new-Schedule::Cron($dispatcher,[extra-args])), the only one we will use is [processprefix](https://metacpan.org/pod/Schedule::Cron#processprefix-=%3E-%3Cname%3E) which is used to give a prefix to the name of our daemon process.

    use Schedule::Cron;
    
    my $cron_daemon = Schedule::Cron->new(
        sub { die "time-daemon: error: default Schedule::Cron function was called\n" },
        processprefix => 'time-daemon'
    );

The most important [Schedule::Cron](https://metacpan.org/pod/Schedule::Cron) method is [add<sub>entry</sub>](https://metacpan.org/pod/Schedule::Cron#$cron-%3Eadd_entry($timespec,[arguments])), which takes a cron string and a coderef. When we eventually run the daemon it will schedule the coderef to be run at the interval specified by the cron string.

Personally I can never remember the syntax for cron strings. I use the website [crontab.guru](https://crontab.guru/) for getting an English translation of what my cron string means which makes it easy to build my cron strings.

Lets schedule our `&append_time` subroutine to be run every 5 minutes, and our `&rotate_time_log` subroutine to be run every hour.

    $cron_daemon->add_entry(
        '*/5 * * * *',
        \&append_time
    );
    
    $cron_daemon->add_entry(
        '0 */1 * * *',
        \&rotate_time_log
    );

We are now ready to start up the daemon using the [Schedule::Cron::run](https://metacpan.org/pod/Schedule::Cron#$cron-%3Erun([options])) method. This method takes many options but the only one we will use is [detach](https://metacpan.org/pod/Schedule::Cron#detach) which will cause daemon process to detach itself from the current process.

    my $pid = $cron_daemon->run(detach => 1);
    
    print "started the time-daemon as pid $pid\n";

That's all there is to it! The basic recipe to follow is: initialize a [Schedule::Cron](https://metacpan.org/pod/Schedule::Cron) object, schedule subroutines to be run with [add<sub>entry</sub>](https://metacpan.org/pod/Schedule::Cron#$cron-%3Eadd_entry($timespec,[arguments])), then start the daemon with the [run](https://metacpan.org/pod/Schedule::Cron#$cron-%3Erun([options])) method.
