The filter module
=================

History
-------

This module started with a simple idea. I wanted an environment
    
    \startmarkdown
    ...
    \stopmarkdown

to write content in Markdown. Such an environment requires a parser that
converts markdown to TeX. A TeX wizard would write such a parser in TeX (which
after all is Turing complete). A lua wizard would write it in LPEG (and wonder
why some users still use MkII). I, being no wizard myself, found an existing
tool that does the job---pandoc. But how could I use pandoc inside ConTeXt?

As for almost everything else in ConTeXt, Hans had already done this; albeit for
a different tool---the R programming language.  The _R
module_ (`m-r.tex`) allows the user to execute snippets of R code. I wanted the
same for Markdown.

I copied the R-module, made a couple of changes, and had a Markdown module
ready. But what if tomorrow I wanted to use ReStructured Text instead of
Markdown? Or Matlab code instead R? Surely, copying the R-module for each
program would be a waste of effort. Each new program requires only a few changes
in the R-module; what I needed was a R-module _template_ so that I could fill in
the blanks with the appropriate values. And so, the filter module was born.

Compatibility
------------

This module works with both MkII and MkIV

Installation
------------

Writing installation instructions is always boring. If you are using ConTeXt
standalone, you can install the module using

    first-setup.sh --modules="t-filter"

Depending on your TeX distribution, you may already have the module.
To verify, check if

    kpsewhich t-filter.tex

returns a meaningful path. If not, you have to manually install the module.

Download the latest version of the module from
[http://github.com/adityam/filter/downloads](http://github.com/adityam/filter/downloads)
and unzip it either `$TEXMFHOME` or `$TEXMFLOCAL`. Run

    mktexlsr

and

    mtxrun --generate

to refresh the TeX file database (for MkII and MkIV, respectively). If
everything went well

    kpsewhich t-filter

will return the path where you stored the file.

Unfortunately, that is not enough. For the module to work, TeX must be able to
call an external program. This feature is a potential security risk and is
disabled by default on most TeX distributions. To enable this feature, you must
set

    shell_escape=t

in your `texmf.cnf` file. See this page
[http://wiki.contextgarden.net/write18](http://wiki.contextgarden.net/write18)
on the ConTeXt wiki for detailed instructions.

Basic Usage
-----------

The steps involved in calling a filter on the contents of an evironment are:

1. Write the contents to an external file. This file is the input to the filter,
   and is, therefore, called `\externalfilterinputfile`

2. Run the filter on `\externalfilterinputfile` to generate an output, which is
   called `\externalfilteroutputfile`.

3. (Optional) Read `\externalfilteroutputfile` in ConTeXt.

Lets start from the simplest case: a filter that inputs a text file and outputs
a valid ConTeXt file, like `pandoc` to convert from Markdown to ConTeXt. The
command line syntax of this filter is

    pandoc -t context -o outputfile inputfile

Using this filter from within ConTeXt is pretty simple:

    \usemodule[filter]

    \defineexternalfilter
        [markdown]
        [filtercommand={pandoc -t context -o \externalfilteroutputfile\space \externalfilterinputfile}]

Yes, its that easy! The only thing to note is that TeX macros gobble spaces, so
we have to manually insert a space after `\externalfilteroutputfile`.

This defines three things:

1. An environment

        \startmarkdown
          ...
        \stopmarkdown

   The contents of the environment are processed by `pandoc` and the output is
   included back in ConTeXt.

2. A macro

        \inlinemarkdown{...}

   The argument of the macro is processed by `pandoc` and the output is included
   back in ConTeXt.

3. A macro

        \processmarkdownfile{...}

   The argument to the macro is a filename, which is processed by `pandoc` and
   the output is included back in ConTeXt.

Dealing with slow filters
-------------------------

The above definition of a markdown filter creates two additional files: an
"input" file and an "output" file, *irrespective of the
number of times the environment is called*. For each markdown environment,
ConTeXt overwrites the input file and pandoc overwrites the output file. This
behavior is the default as I do not want to clutter the current directory with
temporary files. The trade off is that for each document run, the filter is
invoked as many times as the number of markdown environments. Since getting
cross-referencing right normally takes two or three runs, effectively the filter
is run two or three times more than required. A filter like `pandoc` is fairly
fast, so these extra runs are not noticeable. But some filters, like the
R-programming language for which simply startup and exit takes about 0.3
seconds, are slow. In such cases, the additional runs start adding up. A better
trade off is to store the contents of each environment in a separate file and
invoke the filter only if a file *changes in between successive runes*.

The second behavior is achieved by adding `continue=yes` option to the
definition:

    \defineexternalfilter
        [...]
        [...
         continue=yes,
         ...]

Sometimes you want to force a rerun of all filters, even when `continue=yes` is
set. This could be because the filters depend on an external script that might
have changed. To force a rerun of all filters, enable the
[mode](http://wiki.contextgarden.net/Modes) `force` either by adding
`mode=force` to the compiler:

    context --mode=force filename

or adding

    \enablemode[force]

somewhere near the top of your file.

Reading the input
----------------

By default, after the filter is executed, `\externalfilteroutputfile` is read
using `\ReadFile`. To change this behavior, use the `readcommand` option. For
example:

    \defineexternalfilter
      [...]
      [....
       readcommand=\typefile,
       ...]

types the output file verbatim. The value of read command must be a macro that
takes the name of the output file as a (brace-delimited) argument and does
something sensible with it. 

Sometimes, it is desirable to ignore the output, which is done by

    \defineexternalfilter
      [...]
      [....
       read=no,
       ...]


Names of temporary files
------------------------

By default, `\externalfilterinputfile` is set to `\jobname-temp-<filter>.tmp`, where
`<filter>` is the first argument of `\defineexternalfilter`. When `continue=yes`
is set, `\externalfilterinputfile` equals `\jobname-temp-<filter>-<n>.tmp`, where
`<n>` is the number of filter environments that have appeared so far. In this
case,  a `\jobname-temp-<filter>-<n>.tmp.md5` file, which stores the `md5` sum of the
input file is also created.

A macro `\externalfilterbasefile` stores the name of the input file without the
extension. By default, the value of `\externalfilteroutputfile` is
`\externalfilterbasefile.tex`. Having a `.tex` extension is not always
desirable. For example, if the filter generates a PNG file, a `.png` extension
is more descriptive. The name of the output file is changed using the `output`
key. For example

    \defineexternalfilter
        [...]
        [filtercommand={...},
         output={\externalfilterbasefile.png}]

changes the output extension to `.png`. We also need to either set

    \defineexternalfilter
      [...]
      [....
       read=no,
       ...]

or set 

    \defineexternalfilter
      [...]
      [....
       readcommand=\readPNGfile,
       ...]

where `\readPNGfile` is defined as

    \def\readPNGfile#1{\externalfigure[#1]}

Output Directory
----------------

This module creates a lot of temporary files that clutter the current directory.
If you prefer the temporary files to be created in another directory, specify
the `directory` option, e.g.,

    \defineexternalfilter
      [...]
      [...
       directory=output/,
      ...]

This will create all the temporary files in `output` directory. The name of the
directory may be specified with or without a trailing slash. Thus,
`directory=output` and `directory=output/` are both valid. 

The directory path **must be relative to the current directory**. Absolute paths
do not work. If you try to use a absolute path like

    \defineexternalfilter
      [...]
      [...
       directory=/tmp/,
      ...]

you will get an error message

    t-filter        > Fatal Error: Cannot use absolute path /tmp/ as directory

and compilation will stop.

Disabling filters
----------------

Adding `state=stop` option disables the filters. The
`\externalfilterinputfile` is still written, but the filter is not run.

When used in conjunction with `continue=yes` and `directory=...`, this
is useful for sharing your files with others who do not have the
external program that you are using. 

Enabling the `reuse` mode (before the `filter` module is loaded) sets
`state=stop` as the default value.

Deleting temporary files
------------------------

To delete the temporary files generated by the module, call

    context --purgeall --pattern=filename

Do **not** use the file extension. To delete all temporary files in the
current directory, use

    context --purgeall


Standard options
---------------

`\defineexternalfilter` accepts the following standard options:

- `before` and `after`: to set the spacing of the environment or enclose the
  output in a frame, etc. 
- `style` and `color` (these currently only work in MkIV): to set the color and
  style of the output.
- `indentnext`: Should the next line be indented?
- `setups`: specify a list of setups (defined using `\startsetups`). These
  setups may be used to define commands that are needed inside the environment.

The order in which these options are executed are:

1. `before`
2. `style` and `color`
3. `setups`
4. `readcommand`
5. `after`
6. check `indentnext`

Options to a specific environment
---------------------------------

Each `\start<filter>` macro also accepts options. However, unlike other ConTeXt
environment, these options cannot be on a separate line; they must be on the same
line as `\start<filter>`. For example, suppose I define an environment to run
R-code

    \defineexternalfilter
      [R]
      [filtercommand={R CMD BATCH -q  \externalfilterinputfile\space \externalfilteroutputfile},
       output=\externalfilterbasefile.out,
       continue=yes]

I can hide the output of a particular R-environment by

    \startR[read=no]
    ...
    \stopR


A setup to control them all
---------------------------

The macro `\setupexternalfilters` sets the default options for all the filters
created using `\defineexternalfilter`. This is responsible for the default values
of all options. The current defaults are

    \setupexternalfilters
      [before=,
       after=,
       setups=,
       continue=no,
       read=yes,
       readcommand=\ReadFile,
       output=\externalfilterbasefile.tex,
      ]


Passing options to filters
--------------------------

Sometimes it is useful to pass options to a filter. For example, `pandoc`
converts many different formats to ConTeXt (actually, to many different output
formats, but that is irrelevant here). Instead of defining a separate
environment for each input format, can I define a universal pandoc environment
and specify the input format on a case by case basis. For example,

    \startpandoc
    ...
    \stoppandoc

for the default Markdown input,

    \startpandoc[format=rst]
    ...
    \stoppandoc

for reStructured Text input, and

    \startpandoc[format=latex]
    ...
    \stoppandoc

for LaTeX input. In `pandoc`, the input format is specified as

    pandoc -f format -t context -o outputfile inputfile

So, we need a mechanism to access the value of the `format` option to
`\startpandoc`. This value is accessed using `\externalfilterparameter{format}`.
Thus, the pandoc environment may be defined as

    \defineexternalfilter
      [pandoc]
      [filtercommand={pandoc -f \externalfilterparameter{format} -t context 
                       -o \externalfilteroutputfile\space \externalfilterinputfile},
       format=markdown]


Macro variant
-------------

For some cases, a macro `\inline<filter>{...}` is more natural to use rather
than the environment `\start<filter>` ... `\stop<filter>`. The `\inline...`
variant is meant for simple cases, so it does not accept any options in square
brackets. This macro is similar to `\type` macro, and its argument can be
written in two ways: either as a group `{...}` or delimited by arbitrary tokens.
Thus, all the following are valid:

    \defineexternalfilter[markdown][...]

    \inlinemarkdown{both braces{}}

    \inlinemarkdown+an opening brace {+

    \inlinemarkdown!a closing brace }!



Processing Existing Files
-------------------------

A big advantage of a lightweight markup language like markdown is that it is
easy to convert it into other markups--html, rtf, epub, etc. For that reason, I
key in markdown in a separate file rather in a start-stop environment of a TeX
file. To use such markdown files in ConTeXt, I can just use

    \processmarkdownfile{filename.md}

The general macro is `\process<filter>file{...}`, which takes the name of a file
as an argument and uses that file as the input file for the filter. The rest of
the processing is the same as with `\start<filter>` ... `\stop<filter>`
environment. 

The `\process<filter>file` macro also takes an optional argument for setup
options:

    \process<filter>file[...]{...}

The options in the `[...]` are the same as those for `\defineexternalfilter`.

Prepend and append text
-----------------------

**NOTE** Only works in MkIV

Some external programs require boilerplate text at the beginning and end of each
file. Including this boilerplate code in each snippet can get verbose. The
filter module provides two options `bufferbefore` and `bufferafter` to shorten
such snippets. Define the boilerplate code in ConTeXt buffers and then use

    \defineexternalfilter
        [...]
        [...
         bufferbefore={...list of buffers...},
         bufferafter={...list of buffers...},
        ]

For example, suppose you want to generate images using a latex package that does
not work well with ConTeXt, say shak. One way to use this is as follows: first
define a file that processes its content using `latex`.

        \defineexternalfilter
            [chess]
            [filter=pdflatex,
             output=\externalfilterbasefile.pdf,
             readcommand=\readPDFfile,
            ]

        \def\readPDFfile#1{\externalfigure[#1]}


Next create buffers containing boilerplate code needed to run latex:

       \startbuffer[chess::before]
        \documentclass{minimal}
        \usepackage{skak}
        \usepackage[active,tightpage]{preview}

        \begin{document}
        \begin{preview}
        \newgame
        \hidemoves{
      \stopbuffer

      \startbuffer[chess::after]
        }
        \showboard
        \end{preview}
        \end{document}
      \stopbuffer

and tell the filter to prepend and append these buffers

      \setupexternalfilter
        [chess]
        [bufferbefore={chess::before},
         bufferafter={chess::after}]

Then you can use

      \inlinechess{1.e4 e5 2. Nf3 Nc6 3.Bb5}

to get a chess board.


Limitations
------------

- The option `continue=yes` does not work correctly with filters that have a
  pipe `|` in their definition. This is because internally `continue=yes` calls

      mtxrun --ifchanged=filename --direct filtercommand

  and this produces
  
      MTXrun |
      MTXrun | executing: filtercommand
      MTXrun |
      MTXrun |


Messages and Tracing
-------------------

The filter module outputs some diagnostic information on the console output to
indicate what is happening. Loading of the module is indicated by:

    loading         : ConTeXt User Module / Filter

Whenever a filter is executed, the expanded name of the command is displayed.
For example, for the markdown filter we get:

    t-filter        > command : pandoc -w context -o markdown-temp-markdown.tex markdown-temp-markdown.tmp

If, for some reason, the output file is not generated, or not found, a message
similar to

    t-filter        > file markdown-temp-markdown.tex cannot be found
    t-filter        > current filter : markdown
    t-filter        > base file : markdown-temp-markdown
    t-filter        > input file : markdown-temp-markdown.tmp
    t-filter        > output file : markdown-temp-markdown.tex

is displayed on the console. At the same time, the string 

    [[output file missing]]

is displayed in the PDF output. This data, along with the name of the filter
command, is useful for debugging what went wrong. To get more debugging
information add 

    \traceexternalfilters

in your tex file. This shows the name of the filters when they are defined.


Version History
--------------

- **2010.09.26**: 
    - First release
- **2010.10.09**: 
    - Added `\inline<filter>{...}` macro 
    - Changed the syntax of `\process<filter>file`. The filename is now
      specified in curly brackets rather than square brackets.
- **2010.10.16**:
    - Added `\traceexternalfilters` for tracing
    - Added a message that shows filter command on console
    - A message is shown on console when output file is not found.
- **2010.10.30**:
    - Added `directory=...` option to `\defineexternalfilter` and
      `\setupexternalfilters`.
- **2010.12.04**:
    - Bugfix in `directory` code. The option `directory=../something` was
      handled incorrectly.
- **2011.01.28**
    - Bugfix. The filter counter was not incremented inside a group. Made the
      increment global.
- **2011.02.21**
    - Added `style` and `color` options.
- **2011.02.27**
    - The external files are called `\jobname-temp-<filter>*` instead of
      `\jobname-externalfilter-<filter>*`. As a result, these files are deleted
      by `context --purgeall`.
- **2011.03.06**
    - Complete rewrite of internal macro names. The internal macros are now
      named `\modulename::command_name`. This is an experiment to see if this
      style works better than the traditional naming convention in TeX.
- **2011.06.16**
    - Added `force` mode to force recompilation of all filters that have
      `continue=yes`. 
    - Added `reuse` mode to skip running all filters that have
      `continue=yes`. 
    - Added `state=stop` option to skip running external filter.
