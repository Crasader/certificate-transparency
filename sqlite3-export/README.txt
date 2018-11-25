Exporting SQLite to Git
=======================

Theoretically it's as simple as:

  > `fossil export --git | git fast-import`

Unfortunately it doesn't work that way and that's what this project is all
about.


Quick Start
-----------

1. Run the script `build` to fetch and build a suitable fossil tool and a
   `git-export-filter` tool.

2. Run the script `import` (maybe with the `--notes` option)
   to create an `sqlite.git` Git clone of the <http://sqlite.org/src> fossil
   sources.  (May take up to 60 minutes.)

3. See the "Building" section at the bottom of this README to "make" SQLite.


Fossil Issues
-------------

There are two problems with fossil export:

1. fossil versions starting with 1.18 mangle export branch and tag names to
   avoid including characters git does not allow.  The problem is that many
   more characters are mangled than needed so that a tag like `version-1.18`
   is converted to `version_1_18` unnecessarily.

2. fossil versions after 1.18 produce a Git fast-import data stream that
   causes `git fast-import` to fail with a fatal error.


The Tag Problem
---------------

The fossil change that introduces tag mangling is here:

  > <http://www.fossil-scm.org/index.html/info/b707622f29>

It was a well-intentioned change as previously invalid Git names would be
exported, but it went way, way too far.  In fact, the actual Git rules about
allowable characters in names are:

  1. Characters with an ASCII value less than or equal to 32 are not allowed
  2. The character with an ASCII value of 0x7F is not allowed
  3. The characters '~', '^', ':', '\', '*', '?', and '[' are not allowed
  4. The character '/' is a separator and is not allowed within a name
  5. The name may not start with '.' or end with '.'
  6. The name may not end with '.lock'
  7. The name may not contain the '..' sequence
  8. The name may not contain the '@{' sequence
  9. If multiple components are used (separated by '/'), no empty '' components

A patch is included in the file `patches/export_c_patch_diff.txt` that allows
the full diversity of git names to be used and should be applied to the fossil
`src/export.c` file of fossil version 2.1 before building fossil.  It also adds
an optional --notes option to the fossil export --git command that if given
will add a note in the refs/notes/fossil namespace to each commit giving the
original fossil check-in hash for the commit.  Furthermore, it also provides a
new --use-done-feature option (see `git help fast-import`) and makes sure there
aren't any whitespace issues with commit messages by transforming CRLF into
just LF and making sure the only whitespace at the end is a single LF.

There may be updates coming to the official fossil release to address this
name mangling problem, but as of fossil 2.1 they have yet to make it into any
official fossil release.


The Export Problem
------------------

The fossil change that introduces the export problem is here:

  > <http://www.fossil-scm.org/index.html/info/bc8d368b66>

There is even a ticket about this "timewarp" export issue here:

  > <http://www.fossil-scm.org/index.html/info/4013b0a81a>

This issue affects the `sqlite`, `sqlite_docsrc` and `fossil` repositories
making it impossible to export them from fossil and import them into Git with
a current version of fossil.


Let's do the Time Warp Again
----------------------------

The fossil ticket linked to in the above "The Export Problem" section talks
about "timewarps".  These are simply check-ins with a timestamp that is earlier
than at least one of their parents (merges have two parents, most others one).

Fossil doesn't much like these.  The Git fast-import format is a "streamy"
format that, while it allows back references to things earlier in the stream,
does not allow forward references to future, prospective data.  Fossil likes
to output its fast-import stream in check-in date order.  And there you see
the issue.  If a "timewarp" is present then children get put out before their
parents arrive, and Git rudely ends the fast-import operation when this occurs.

All three of the primary fossil repositories (SQLite, SQLite Docs, Fossil) have
at least one "timewarp" in them.

Fossil versions 1.18 and earlier produce a usable fast-import stream not
because it orders the output check-ins correctly in spite of the "timewarp",
but because it outputs all data for each check-in rather than outputting only
differences from the parent(s).  So while the output isn't really correct, it
is accepted by Git and when outside the "timewarp" portion of the history,
the converted Git commits have exactly the correct set of sources, so it's
really not much more than a minor annoyance when reviewing very tiny parts
of older history in the repository.

Starting with fossil version 1.19 this all changed.  Now, whenever possible,
the exported Git fast-import stream only includes "changes" from a check-in's
parent(s).  With a sloppy ordering based only on check-in timestamp and in
the presence of "timewarps", children get put out before their parent(s) arrive
with the ensuing Git rudeness.  While, on the surface, this seems like a good
change (and it brought the ability to do incremental exports), full exports
seem to take somewhat longer overall now.

Then on 2017-02-23, they "shattered it" <https://shattered.it/>.

Shortly thereafter fossil version 2.0 came out supporting additional hash
functions.  And on 2017-03-12 the official SQLite fossil repository got its
first check-in using the new hash function.  Versions of fossil prior to 2.0
cannot deal with these new hash function values.

Now you see the problem.  Fossil version 1.18 can no longer be used (even with
its technically incorrect output) as it cannot understand the new hash values.
But fossil versions 1.19 and later (including 2.0) cannot be used either since
they produce a completely unacceptable fast-import stream in the presence of
any "timewarps".

But, curiosity is a harsh mistress.  The topological ordering problem was
solved even for fossil 1.18 in a satisfactory way some time ago but never
published to avoid causing all the Git refs values to be force-updated.
Correcting the misordering caused by the "timewarps" alters the DAG (directed
acyclic graph) of check-in ancestry and that trickles down to all the children
causing all of their `commit` hash values to change even though the sources they
refer to remain completely unchanged.

As of 2017-03-12 there really isn't a choice anymore.

A GPL version 2 (or later) patch is included to address this in the file
`patches/export_topo_patch_diff.txt` that provides a guaranteed topological
ordering to the exported fast-import stream.  When it's built into fossil,
that version of fossil becomes also covered by the GPL.  The repository data
fossil maintains is unaffected by fossil's license(s) so having a GPL-covered
fossil binary should not really affect anyone.


Fossil Sources
--------------

A .tar.gz archive of the fossil 2.1 sources may be fetched from:

  > <http://fossil-scm.org/index.html/uv/fossil-src-2.1.tar.gz>
  
The downloaded .tar.gz file should have these size and hash values:

  > size:   4802504 bytes  
  > md5:    9f32b23cecb092d42cdf11bf003ebf8d  
  > sha1:   7c7387efb4c0016de6e836dba6f7842246825678  
  > sha256: 85dcdf10d0f1be41eef53839c6faaa73d2498a9a140a89327cfb092f23cfef05

The `archives` subdirectory contains a copy of this .tar.gz file and it will
be used by the build script to create a `fossil` executable that reports its
version as `2.1+export ` to confirm that it contains the export fixes.


Git Fast Import Issues
----------------------

The Git fast-import facility does not provide a means to filter the incoming
data stream to adjust user names (fossil export data only includes the user
login name as the email address) nor a means to adjust branch/tag names
(fossil exports a 'trunk' branch where Git expects a 'master' branch and fossil
also exports what are essentially lightweight tags as annotated tags).

To deal with these issues, the `git-export-filter` utility is used.

It can be found at:

  > <http://repo.or.cz/w/git-export-filter.git>

The included `sqlite_authors` file is used with the `git-export-filter` tool to
supply real user names and email addresses.  Also note that the `sqlite_authors`
file also works for the <http://sqlite.org/docsrc> fossil repository as well.


The Final Solution
------------------

After building a patched version of fossil 2.1 as described above and the
`git-export-filter` utility, a Git repository of the SQLite sources can be
created like so (which is what the `import --notes` script does):

  >     fossil clone http://sqlite.org/src sqlite.fsl
  >     git --git-dir=sqlite.git init --bare
  >     fossil export --git --notes sqlite.fsl |
  >     git-export-filter --authors-file sqlite_authors --require-authors \
  >       --trunk-is-master --convert-tagger tagger |
  >     git --git-dir=sqlite.git fast-import

The above will create the `sqlite.git` Git repository that is a clone of the
SQLite sources from the SQLite fossil respository <http://sqlite.org/src>
(note that only sources are cloned, not tickets or wiki pages or events).

The provided `build` script will attempt to download the necessary sources,
patch them and build suitable `fossil` and `git-export-filter` executable files.

The provided `import` script will then attempt to clone the SQLite sources
and convert them into an `sqlite.git` repository.  It may be run again to update
the `sqlite.git` repository with new changes.  It accepts the `--notes` option
(which is recommended) to enable generation of the `refs/notes/fossil` notes
containing the original fossil check-in hash.

The initial run of the `import` script may take up to 60 minutes on a fast
machine, and subsequent runs of `import` even on a fast machine will still,
unfortunately, take some time.  The CPU will be pounded in either case.

**IMPORTANT**

Options passed to the `import` script are *not* remembered, so make sure to
pass the same options, (e.g. `--notes`) to the `import` script every time it's
run if it's being used to update a previously exported Git repository or you
may end up with out-of-date notes.


Options
-------

There are new options provided by the patch files for the `fossil export`
command.  As a convenience, they may be given to the `import` script which
will just pass them on to the `fossil export` command.

`--notes`

>   Included with the export tags patch a new fossil export option `--notes` is
>   provided that adds a Git commit note to the `refs/notes/fossil` namespace
>   which contains the original fossil check-in hash for each fossil checkin
>   exported to Git.  Use `git log --notes=fossil` to see these notes.

`--use-done-feature`

>   Included with the export tags patch a new fossil export option
>   `--use-done-feature` is provided that includes the `feature done` and
>   `done` commands at the beginning and end respectively of the exported
>   fast-import stream.  This can help avoid partial imports.  See the
>   `git help fast-import` description of the ``--done`` option and the
>   `git help fast-export` description of the ``--use-done-feature`` option.


Building
--------

**QUICKLY**

 1. Clone/checkout the new `sqlite.git` repository into a new working tree

 2. Run the `create-fossil-manifest` script from this repository with the
    current working directory set to the new working tree created in (1)

 3. Now run the `configure` script in the new working tree created in (1)

 4. Now run `make` in the new working tree created in (1)

**DETAILS**

Ideally, simply cloning from the new `sqlite.git` repository would allow one to
then build SQLite by simply using `make` (or `configure` and `make`).

Unfortunately, this is not the case, the make will fail with a message about
no rule to make the files `manifest` and/or `manifest.uuid`.

Both the SQLite sources and the Fossil sources require two fossil vcs specific
files to be created (`manifest` and `manifest.uuid`) in order for make to be
successful.

The `manifest.uuid` file simply contains the hash of the current checkout
and while a real `manifest` file contains a bunch of information, the only
thing that need be present is a line containing the UTC ISO date preceded
by 'D '.

The `create-fossil-manifest` script takes care of creating these files and
should be run with the current working directory set to the top-level of the
git clone's working directory.

Any time the HEAD commit changes, the `create-fossil-manifest` script should
be run to update the `manifest` and `manifest.uuid` files before next running
make or the output of the `sqlite_source_id()` function will be incorrect.
