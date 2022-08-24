Code that performs side effects is difficult to test because we need figure out how to sandbox the effects so we can observe the state of the sandbox before and after executing the effectful code. The difficulty is increased when the side effectful code also depends on specific OS configurations. Let us explore my solution to such a predicament.

I have been working on the next major release of my [btrfs](https://btrfs.wiki.kernel.org/index.php/Main_Page) snapshot manager [yabsm](https://github.com/NicholasBHubbard/yabsm) and I want to write unit tests for functions that take and delete btrfs snapshots. This code performs the side effect of taking and deleting snapshots and depends on the OS having a btrfs subvolume available that the user running the program has read+write permissions on.

Yabsm is written in Perl so if you don't know Perl it may be difficult to follow the code examples.


<a id="org8ffeacf"></a>

# Disclaimer

This is just a description of a solution to a problem I came across. I do not claim to be any kind of authority on code testing.


<a id="orga1cad11"></a>

# A quick note on btrfs

Btrfs is a Linux filesystem that allows you to take snapshots of your filesystem. A btrfs filesystem is organized into various "subvolumes" that can be mounted at various locations in your file tree. A common configuration is to have three subvolumes mounted at `/`, `/home`, and `/.snapshots` so you can seperately snapshot your `/`, and `/home` directories, and store the snapshots in `/.snapshots`.


<a id="orga6ee1a8"></a>

# The code to be tested

Let us assume we have already defined the following 4 predicates.

`is_btrfs_subvolume` is satisfied if passed a string representing the path of a btrfs subvolume on the system.

`is_btrfs_dir` is satisfied if passed a string representing a directory on the system that resides on a btrfs subvolume.

`is_btrfs_snapshot` is satisfied if passed a string representing a path to a btrfs snapshot on the system. This predicate is a bit of a fib because every snapshot is also a subvolume and thus would also be satisfied by `is_btrfs_subvolume`. For simplicity purposes we will pretend that we can differentiate between subvolumes and snapshots.

`can_read_write_dir` is satisfied if passed a directory that the current user has read+write permissions for.   

    sub take_snapshot {
    
        # Take a read-only btrfs snapshot of $subvolume named $name and place it in
        # $destination.
    
        my $name        = shift;
        my $subvolume   = shift;
        my $destination = shift;
    
        # preconditions
        return 0 unless is_btrfs_subvolume($subvolume);
        return 0 unless can_read_write_dir($subvolume);
        return 0 unless is_btrfs_dir($destination);
        return 0 unless can_read_write_dir($destination);
    
        my $cmd    = "btrfs subvolume snapshot -r '$subvolume' '$destination/$name'";
        my $status = system $cmd;
    
        unless (0 == $status) {
            die "Aborting because '$cmd' exited with non-zero status";
        }
    
        return 1;
    }
    
    sub delete_snapshot {
    
        # Delete the btrfs snapshot $snapshot.
    
        my $snapshot = shift;
    
        # preconditions
        return 0 unless is_btrfs_snapshot($snapshot);
        return 0 unless can_read_write_dir($snapshot);
    
        my $cmd    = "btrfs subvolume delete '$snapshot'";
        my $status = system $cmd;
    
        unless (0 == $status) {
            die "Aborting because '$cmd' exited with non-zero status";
        }
    
        return 1;
    }


<a id="org2b68600"></a>

# Testing the code

As you can see the code above uses the 4 predicates to assert that preconditions are met before we perform the actual side effect of taking or deleting a snapshot. It is also important to notice that if the side effect fails (determined via `btrfs`'s exit status) then we kill the program. There is an underlying assumption going on here; if certain preconditions are met then we can be sure that our `btrfs` system command will run successfully.

Hmm, maybe in our test environment we can set up different scenarios around these preconditions and see if our assumptions are correct.

1.  Finding a btrfs subvolume

    We cannot take and delete snapshots unless we have a btrfs subvolume available. The simplest way to find a btrfs subvolume is to ask the tester to supply us one via a command line parameter. We can use Perl's built-in [Getopt::Long](https://perldoc.perl.org/Getopt::Long) library to make this easy.
    
        use Getopt::Long;
        my $BTRFS_SUBVOLUME;
        GetOptions( 's=s' => \$BTRFS_SUBVOLUME );
    
    We now have a variable `$BTRFS_SUBVOLUME`, that if defined means the tester supplied us with a btrfs subvolume.
    
    Perl's built-in [Test::More](https://perldoc.perl.org/Test::More) library allows us to skip tests if certain conditions are met so we can use the definedness of `$BTRFS_SUBVOLUME` for such conditions.    

2.  Setting up the sandbox

    If `$BTRFS_SUBVOLUME` is defined then we can attempt to set up our sandbox.
    
    We will use the `tempdir` function from the built-in `File::Temp` library to create a sandbox directory that will be removed when our test script terminates. This sandbox will reside on the `$BTRFS_SUBVOLUME` which means we can place snapshots inside it.
    
    We will require that our test script needs to be run with root privilages so we can be sure we have the necessary permissions for taking and deleting snapshots.
    
        use File::Temp 'tempdir';
        
        my $BTRFS_SANDBOX;
        if ($BTRFS_SUBVOLUME) {
            die "Must be root user" if $<;
            die "'$BTRFS_SUBVOLUME' is not a btrfs subvolume" unless is_btrfs_subvolume($BTRFS_SUBVOLUME);
            $BTRFS_SANDBOX = tmpdir('sandboxXXXXXX', DIR => $BTRFS_SUBVOLUME, CLEANUP => 1);
            die "'$BTRFS_SANDBOX' is not a btrfs directory" unless is_btrfs_dir($BTRFS_SANDBOX);
        }

3.  Testing

    We are ready to write our tests! Lets use the [Test::Exception](https://metacpan.org/pod/Test::Exception) library from [CPAN](https://www.cpan.org/) to test that our subroutines don't kill the program when they're not supposed to.
    
    Please refer to the documentation on [Test::Exception::lives<sub>and</sub>](https://metacpan.org/pod/Test::Exception#lives_and), [Test::More::is](https://perldoc.perl.org/Test::More#is) and [Test::More SKIP blocks](https://perldoc.perl.org/Test::More#SKIP:-BLOCK) if you are confused about the test framework specific code.
    
    Here's the tests - be sure to read the comments!
    
        use Test::More 'no_plan';
        use Test::Exception;
        
        SKIP: {
            skip "Skipping btrfs specific tests because we don't have a btrfs sandbox available", 9
                unless $BTRFS_SUBVOLUME;
        
            ### take_snapshot
        
            # All the preconditions for taking a snapshot should be met
            lives_and { is take_snapshot('foo', $BTRFS_SUBVOLUME, $BTRFS_SANDBOX), 1 } 'take_snapshot terminated are returned true';
        
            # Make sure the snapshot was actually created
            is(is_btrfs_snapshot("$BTRFS_SANDBOX/foo"), 1, 'The snapshot was created');
        
            ### delete_snapshot
        
            # All the preconditions for deleting a snapshot should be met
            lives_and { is delete_snapshot("$BTRFS_SANDBOX/foo"), 1 } 'delete_snapshot terminated and returned true';
        
            # Make sure the snapshot was actually deleted
            is(is_btrfs_snapshot("$BTRFS_SANDBOX/foo"), 0, 'The snapshot was deleted');
        
            ### Preconditions not met
        
            # There is no subvolume named "$BTRFS_SANDBOX/quux"
            lives_and { is take_snapshot('foo', "$BTRFS_SANDBOX/quux", $BTRFS_SANDBOX), 0 } 'take_snapshot returns false if non-existent subvolume';
            is(is_btrfs_snapshot("$BTRFS_SANDBOX/foo"), 0, 'no snapshot was created');
        
            # There is no btrfs directory named "$BTRFS_SANDBOX/quux"
            lives_and { is take_snapshot('foo', $BTRFS_SUBVOLUME, "$BTRFS_SANDBOX/quux"), 0 } 'take_snapshot returns false if non-existent btrfs target dir;
            is(is_btrfs_snapshot("$BTRFS_SANDBOX/quux/foo"), 0, 'no snapshot was created');
        
            # There is no snapshot named "BTRFS_SANDBOX/quux"
            lives_and { is delete_snapshot("$BTRFS_SANDBOX/quux"), 0 } 'delete_snapshot returns false if non-existent snapshot;
        }
    
    The way I test the code is by testing that if `take_snapshot` and `delete_snapshot` are called with arguments that satisfy their preconditions, the functions execute succesfully. I then then observe the state of the sandbox to see if a snapshot was in fact taken or deleted.
    
    I also test that if I call the functions with arguments that do not satisfy the preconditions then the side-effect of taking/deleting a snapshot is never performed. 


<a id="orge975103"></a>

# Summary

The first step to testing side-effectful code is to write the code in a way that allows it to be tested. I used a set of preconditions on function arguments that if satisfied should result in succesful execution of the side effect. I was able to set up a testing sandbox where I can observe the valididity of these assumptions.
