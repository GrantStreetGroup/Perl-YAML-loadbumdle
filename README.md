# NAME

YAML::LoadBundle - Load a directory of YAML files as a bundle

# VERSION

version v0.0.1

# SYNOPSIS

    use YAML::LoadBundle qw(:all);

    my $hash = load_yaml_bundle( "/path/to/yaml_bundle/dir/" );

# DESCRIPTION

Adds additonal features to YAML::XS to allow splitting a YAML file into
multiple files in a common directory.
This helps with readability when the file is large.

It also provides more advanced merging features than the standard YAML spec.

# Export

Nothing is exported by default, but all the functions listed
below are available for export.

All will be exported with :all.

# Exported Functions

## Load

### load\_yaml

    load_yaml($filename)
    load_yaml(\*FILEHANDLE)
    load_yaml($yaml_data)

Parses a YAML file (or string) with extra error-checking.

When passed a $filename, the %YAML\_cache will cache a dclone'd
copy of the result for later retrieval, unless the file has
been modified since the last load. This can be prevented by
passing anything else as a 2nd parameter, if you know that the
file will never be reloaded.

When passed a string, the %YAML\_cache will cache a copy of the result,
using the SHA1 digest of the string as the hash key.
Strings are only cached in memory, not on disk.

After loading the YAML into a Perl data structure, some postprocessing is done
on specially-named keys in hash references.  Each is merged into their
containing hash reference, though at different times and with different
strategies.

- `-import`

    shallow merge (e.g. `%x = (%y, %x)`)

- `-export`

    shallow merge

- `-merge`

    deep merge (see [Hash::Merge::Simple](https://metacpan.org/pod/Hash::Merge::Simple))

- <-flatten>

        some_key: { -flatten: [*SomeArrayRef, *SomeOtherArrayRef] }

    Instead of doing any kind of special hash merging, this special key takes an
    arrayref of arrayrefs, merges all their contents into one large arrayref, then
    replaces its entire surrounding hash with the arrayref.

    In other words, the example above would look like this in Perl:

        { some_key => [@$some_array_ref, @$some_other_array_ref] }

- `-flattenhash`

        some_key: { -flattenhash: [*SomeHashRef, *SomeOtherHashRef] }

    Sometimes it is desirable to import more than one hash object,
    but cannot due to limitations of keynames like -import.
    If you really want this behavior, add this flag.

`import` and `export` are backwards-compatibility synonyms for `-import`
and `-export`.

Like normal list assignment in Perl, the right-hand side takes precedence
(pseudocode):

    %hash = deep_merge(%merge, shallow_merge(%import, %export, %hash));

Instead of a hash reference, any of these keys may contain an array reference
of hash references, in which case those hash references are merged using
whatever strategy normally applies (e.g. deep merge for `-merge`).

### load\_yaml\_bundle

    load_yaml_bundle($path, \%options)

Similar to ["load\_yaml"](#load_yaml), but loads YAML from a bundle of configuration files.
This may be a single file, a directory containing configuration files,
or a whole directory tree of configuration files.

The given path names the base location for the bundle.
This starts by loading a file with the given name followed by a configuration
prefix (either `.yml` or `.conf` by default).
It then checks to see if there is a directory with the same name as the path.
It then repeats the loading process for all nested files and directories where
the file and directory names become the keys into which the configuration is
injected.

For example, given a directory layout like this:

    conf/base.conf
    conf/base/common.conf
    conf/base/user.conf
    conf/base/user/accounts.conf

A hash would be returned mapped something like this (pseudo-code):

    {
        common => load_yaml("conf/base/common.conf"),
        user   => {
            accounts => load_yaml("conf/base/user/accounts.conf"),
            %{ load_yaml("conf/base/user.conf") }
        },
        %{ load_yaml("conf/base.conf") }
    }

However, the actual merge will be a deep merge.

The usual semantics related to import, export, and merging apply to these files
as they do in ["load\_yaml"](#load_yaml).

Symlinks can be used to share the data between multiple keys.
By default, symlinks will be followed, but will cause an error if any of them
are outside the root path of the bundle.
If a symlink is permitted, this will follow symlinks to files or directories as
if they were the files or directories set locally, allowing the original key to
be renamed in any way desired.
It also means that symlinks to files must have a correct configuraiton file
suffix.

This will ignore any file starting with a period.

There are some options to modify the default behaviors:

- `follow_symlinks_when`

    This may be set to any of the following strings:

    - `bundled`

        This is the default. Symlinks are followed, but only within the root.

    - `never`

        Do not follow symlinks.

    - `always`

        Always follow symlinks. Use this with caution.

- `follow_symlinks_fail`

    This may be set to any of the following strings:

    - `error`

        This is the default. When symlinks are found, but not followed, croak.

    - `warn`

        When symlinks are found, but not followed, carp.

    - `ignore`

        When symlinks are found, but not followed... take no action at all.

- `conf_suffixes`

    This routine only considers files that have the given suffxes. The default includes "conf" and "yml". The suffixes are given as an array reference of strings. (All directories will be followed, at least to the maximum depth.)

- `max_depth`

    This defaults to 20 which is probably more than enough and mostly intended to prevent some sort of insane failure. If a directory is found at one further than the maximum depth, a warning will be issued.

## Observe

### add\_yaml\_observer

    add_yaml_observer(sub {
        my $filename = shift;
        warn "Yaml file $filename was just loaded.";
    });

Adds an observer sub that will be notified just prior to a yaml
file being loaded.  Note that each observer is called even if
the yaml file is cached and does not need to be reloaded.

### \_notify\_yaml\_observers

Called internally to notify each waiting observer
that a new yaml file is being loaded.

### remove\_yaml\_observer

    remove_yaml_observer($subref);

Removes an observer sub that was previously added via add\_yaml\_observer.

## Cache

### $YAML::LoadBundle::CacheDir

Set this to a path that already exists of where to cache loaded files.

    $YAML::LoadBundle::CacheDir
        = File::Spec->catdir( File::Spec->tmpdir, 'yaml-loadbundle' );

Defaults to `$ENV{YAML_LOADBUNDLE_CACHEDIR}`.

If false, caching is disabled.

# AUTHOR

Grant Street Group &lt;developers@grantstreet.com>

# COPYRIGHT AND LICENSE

This software is Copyright (c) 2016 - 2020 by Grant Street Group.

This is free software, licensed under:

    The Artistic License 2.0 (GPL Compatible)
