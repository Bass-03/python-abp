# python-abp

This repository contains a library for working with Adblock Plus filter lists
and the script that is used for building Adblock Plus filter lists from the
form in which they are authored into the format suitable for consumption by the
adblocking software.

## Installation

Prerequisites:

* Linux, Mac OS X or Windows (any modern Unix should work too),
* Python (2.7 or 3.5+),
* pip.

To install:

    $ pip install -U python-abp

## Rendering of filter lists

The filter lists are originally authored in relatively smaller parts focused
on a particular type of filters, related to a specific topic or relevant
for particular geographical area.
We call these parts _filter list fragments_ (or just _fragments_)
to distinguish them from full filter lists that are
consumed by the adblocking software such as Adblock Plus.

Rendering is a process that combines filter list fragments into a filter list.
It starts with one fragment that can include other ones and so forth.
The produced filter list is marked with a [version and a timestamp][1].

Python-abp contains a script that can do this called `flrender`:

    $ flrender fragment.txt filterlist.txt

This will take the top level fragment in `fragment.txt`, render it and save into
`filterlist.txt`.

The `flrender` script can also be used by only specifying `fragment.txt`:
    
    $flrender fragment.txt
   
in which case the rendering result will be sent to `stdout`. Moreover, when 
it's run with no positional arguments:

    $flrender

it will read from `stdin` and send the results to `stdout`.

Fragments might reference other fragments that should be included into them.
The references come in two forms: http(s) includes and local includes:

    %include http://www.server.org/dir/list.txt%
    %include easylist:easylist/easylist_general_block.txt%

The first instruction contains a URL that will be fetched and inserted at the
point of reference. 
The second one contains a path inside easylist repository.
`flrender` needs to be able to find a copy of the repository on the local
filesystem. We use `-i` option to point it to to the right directory:

    $ flrender -i easylist=/home/abc/easylist input.txt output.txt

Now the second reference above will be resolved to
`/home/abc/easylist/easylist/easylist_general_block.txt` and the fragment will
be loaded from this file.

Directories that contain filter list fragments that are used during rendering
are called sources.
They are normally working copies of the repositories that contain filter list
fragments.
Each source is identified by a name: that's the part that comes before ":"
in the include instruction and it should be the same as what comes before "="
in the `-i` option.

Commonly used sources have generally accepted names. For example the main
EasyList repository is referred to as `easylist`.
If you don't know all the source names that are needed to render some list,
just run `flrender` and it will report what it's missing:

    $ flrender easylist.txt output/easylist.txt
    Unknown source: 'easylist' when including 'easylist:easylist/easylist_gener
    al_block.txt' from 'easylist.txt'

You can clone the necessary repositories to a local directory and add `-i`
options accordingly.

## Rendering diffs

A diff allows a client running ad blocking software such as Adblock Plus to
update the filter lists incrementally, instead of downloading a new copy of a
full list during each update. This is meant to lessen the amount of resources
used when updating filter lists (e.g. network data, memory usage, battery
consumption, etc.), allowing clients to update their lists more frequently using
less resources.

Python-abp contains a script called `fldiff` that will find the diff between the
latest filter list, and any number of previous filter lists:

    $ fldiff -o diffs/easylist easylist.txt archive/*

where `-o diffs/easylist` is the (optional) output directory where the diffs
should be written, `easylist.txt` is the most recent version of the filter list,
and `archive/*` is the directory where all the archived filter lists are. When
called like this, the shell should automatically expand the `archive/*`
directory, giving the script each of the filenames separately.

In the above example, the output of each archived `list[version].txt` will be
written to `diffs/diff[version].txt`. If the output argument is omitted, the
diffs will be written to the current directory.

The script produces three types of lines, as specified in the [technical
specification][5]:

* Special comments of the form `! <name>:[ <value>]`
* Added filters of the form `+ <filter-text>`
* Removed filters of the form `- <filter-text>`


## Library API

Python-abp can also be used as a library for parsing filter lists. For example
to read a filter list (we use Python 3 syntax here but the API is the same):

    from abp.filters import parse_filterlist

    with open('filterlist.txt') as filterlist:
        for line in parse_filterlist(filterlist):
            print(line)

If `filterlist.txt` contains a filter list:

    [Adblock Plus 2.0]
    ! Title: Example list

    abc.com,cdf.com##div#ad1
    abc.com/ad$image
    @@/abc\.com/
    ...

the output will look something like:

    Header(version='Adblock Plus 2.0')
    Metadata(key='Title', value='Example list')
    EmptyLine()
    Filter(text='abc.com,cdf.com##div#ad1', selector={'type': 'css', 'value': 'div#ad1'}, action='hide', options=[('domain', [('abc .com', True), ('cdf.com', True)])])
    Filter(text='abc.com/ad$image', selector={'type': 'url-pattern', 'value': 'abc.com/ad'}, action='block', options=[('image', True)])
    Filter(text='@@/abc\\.com/', selector={'type': 'url-regexp', 'value': 'abc\\.com'}, action='allow', options=[])
    ...

`abp.filters` module also exports a lower-level function for parsing individual
lines of a filter list: `parse_line`. It returns a parsed line object just like
the items in the iterator returned by `parse_filterlist`. 

For further information on the library API use `help()` on `abp.filters` and
its contents in interactive Python session, read the docstrings or look at the
tests for some usage examples.

## Testing

Unit tests for `python-abp` are located in the `/tests` directory.
[Pytest][2] is used for quickly running the tests
during development.
[Tox][3] is used for testing in different
environments (Python 2.7, Python 3.5+ and PyPy) and code quality
reporting.

In order to execute the tests, first create and activate development
virtualenv:

    $ python setup.py devenv
    $ . devenv/bin/activate

With the development virtualenv activated use pytest for a quick test run:

    (devenv) $ pytest tests

and tox for a comprehensive report:

    (devenv) $ tox

## Development

When adding new functionality, add tests for it (preferably first). Code
coverage (as measured by `tox -e qa`) should not decrease and the tests
should pass in all Tox environments.

All public functions, classes and methods should have docstrings compliant with
[NumPy/SciPy documentation guide][4]. One exception is the constructors of
classes that the user is not expected to instantiate (such as exceptions).


## Using the library with R

Clone the repo to you local machine. Then create a virtualenv and install 
python abp there:

        $ cd python-abp
        $ virtualenv env
        $ pip install --upgrade .

Then import it with `reticulate` in R:

        > library(reticulate)
        > use_virtualenv("~/python-abp/env", required=TRUE)
        > abp <- import("abp.filters.rpy")

Now you can use the functions with `abp$functionname`, e.g. 
`abp.line2dict("@@||g.doubleclick.net/pagead/$subdocument,domain=hon30.org")`


 [1]: https://adblockplus.org/filters#special-comments
 [2]: http://pytest.org/
 [3]: https://tox.readthedocs.org/
 [4]: https://github.com/numpy/numpy/blob/master/doc/HOWTO_DOCUMENT.rst.txt
 [5]: https://docs.google.com/document/d/1SoEqaOBZRCfkh1s5Kds5A5RwUC_nqbYYlGH72sbsSgQ/
