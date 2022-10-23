[Dist::Zilla](https://dzil.org/) (dzil) is a program for creating Perl distributions. While the documentation for dzil is complete, it is not geared towards a beginner that has never created a Perl distribution before. This article provides a brief introduction to Dist::Zilla geared towards users that know little about Perl distributions in general.


<a id="orgafbbf0c"></a>

# What is a Perl distribution?

A Perl distribution is an archive of files that includes a Perl module. There are no official rules on what non-module files must be included in a distribution, but they often include (among other things) test scripts, a Makefile.PL, documentation, and the license. These distributions are commonly uploaded to [CPAN](https://metacpan.org/), which is a place for Perl programmers to upload their Perl distributions for the purpose of sharing their code.


<a id="orgcabe74a"></a>

# Why Dist::Zilla?

You may think that bundling together a Perl module with some other files is simple, but there are many things that need to be accounted for, and are prone to human error. There are also many possibilities for what somebody may want to include in a distribution, and how they want to include it. Dist::Zilla exists to be a one-stop solution to every possible problem involved in creating a Perl distribution.


<a id="org29a19e9"></a>

# Using Dist::Zilla

When you install Dist::Zilla, you will be provided with an executable named `dzil`. The most important command that `dzil` provides is `build`, which - when run in the projects root directory - outputs a distribution tarball. Other commands such as `test` and `release` are also provided, but when getting started with Dist::Zilla you will only need the `build` command.


<a id="orge0178c3"></a>

# The "dist.ini" File

Dist::Zilla is configured on a per-project basis through a file named `dist.ini`, which should be located at the root of the project's directory tree.

The beginning of a `dist.ini` file specifies required settings that every distribution should have. These settings include `name`, `version`, `abstract`, `copyright_holder`, and `license`. (There is also `author`, which isn't required but you probably want to add it as well.)

Here is an example:

    name = App-Foo
    version = 1.0
    author = Jane Doe
    copyright_holder = Jane Doe
    license = Perl_5
    abstract = the best software ever

After you specify these required settings, you can then configure your distribution by specifying what plugins you wish to use. Plugins are the mechanism that Dist::Zilla uses for providing features to your Perl distribution. If you have `dist.ini` that doesn't specify any plugins, Dist::Zilla will produce an empty distribution with no files.

Let's look at example of the plugins that a simple distribution might use, then go over what a few of the plugins actually do:

    [MetaResources]
    homepage       = https://github.com/JaneDoe/App-Foo
    bugtracker.web = https://github.com/JaneDoe/App-Foo/issues
    repository.url = https://github.com/JaneDoe/App-Foo.git
    
    [GatherDir]
    [PruneCruft]
    [ManifestSkip]
    [MetaYAML]
    [License]
    [ExecDir]
    [MakeMaker]
    [Manifest]
    [AutoPrereqs]
    [TestRelease]

The [MetaResources](https://metacpan.org/pod/Dist::Zilla::Plugin::MetaResources) plugin adds resource entries to the distribution's metadata. [MetaCPAN](https://metacpan.org/) can use this information to provide useful links to the distribution's page.

The [GatherDir](https://metacpan.org/pod/Dist::Zilla::Plugin::GatherDir) and [PruneCruft](https://metacpan.org/pod/Dist::Zilla::Plugin::PruneCruft) plugins tell Dist::Zilla that you want to include all the files in your project's directory into the distribution, excluding the ones you certainly don't want. The files you certainly don't want include build artifacts introduced by recent invocations of Dist::Zilla. The combination of these two plugins is used in almost every Dist::Zilla project.

The [MakeMaker](https://metacpan.org/pod/Dist::Zilla::Plugin::MakeMaker) plugin will tell Dist::Zilla to produce an [ExtUtils::MakeMaker](https://metacpan.org/pod/ExtUtils::MakeMaker)-powered Makefile.PL. Dist::Zilla will deal with everything required to create a proper Makefile.PL, so you do not need to know anything about ExtUtils::MakeMaker. Unless you are doing something special, you almost certainly want to use this plugin.

The [UploadToCPAN](https://metacpan.org/pod/Dist::Zilla::Plugin::UploadToCPAN) plugin will allow you to use the `dzil release` command to upload your distribution to CPAN.

It is important to note that each plugin takes effect in the order the plugins are specified in your dist.ini.

There are **many** plugins available for Dist::Zilla - over 1,200 thus far - so you will probably find one that can do just about anything you could possibly need for creating a distribution. [Here](https://metacpan.org/search?size=20&q=Dist%3A%3AZilla%3A%3APlugin) is a link for a metacpan query for "Dist::Zilla::Plugin", that can be used to explore the Dist::Zilla plugin ecosystem.

Here are links to the documentation for the plugins in the example `dist.ini` that I did not explain:

-   [ManifestSkip](https://metacpan.org/pod/Dist::Zilla::Plugin::ManifestSkip)
-   [MetaYAML](https://metacpan.org/pod/Dist::Zilla::Plugin::MetaYAML)
-   [License](https://metacpan.org/pod/Dist::Zilla::Plugin::License)
-   [ExecDir](https://metacpan.org/pod/Dist::Zilla::Plugin::ExecDir)
-   [Manifest](https://metacpan.org/pod/Dist::Zilla::Plugin::Manifest)
-   [AutoPrereqs](https://metacpan.org/pod/Dist::Zilla::Plugin::AutoPrereqs)
-   [TestRelease](https://metacpan.org/pod/Dist::Zilla::Plugin::TestRelease)


<a id="org23f3c96"></a>

# Synopsis

Dist::Zilla can seem daunting at first, but it is actually quite straightforward and easy to use once you figure it out. The only difficult thing is figuring out what plugins you want to use.
