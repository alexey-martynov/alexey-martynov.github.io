---
layout: post
title: Complex LaTeX Usage
permalink: /posts/complex-latex-usage.html
---

[LaTeX](https://www.latex-project.org/) is a very sophisticated
document preparation system. It makes possible to create fine
documents with a lot of technical stuff like formulas, diagrams and
code listings in a relatively easy way. Accompanied with bibliography
manager (BibLaTeX or Biber) and index generator LaTeX outperforms
standard office suites. In this article I'll show how to maintain set
of related documents which evolve over the time using
LaTeX.

Everything in this article is applicable to PDF LaTeX or
XeLaTeX.

> Disclaimer: this article contains personal vision which might not
> match somebody else vision. Also it contains some value
> judgements.
>
> This article can't and doesn't try to replace LaTeX guides. Please
> refer to such guides and package's reference documents for an extra
> details about commands mentioned.

## Table of Contents
{:.no_toc}
* toc
{:toc}

## A Document Set Example

Lets take as example an university course. The good university might
have a set of related documents aside from bureaucratic requirements:

* A textbook which contains all information in course. It may contain
  much more information than used in lectures because it is created
  for offline usage.

* A set of presentations used during lectures. It is tiresome and time
  consuming to write on board, for example, sample programs. Such
  written sample have small typos very often and may confuse
  inexperienced students.

* A set of handouts from lectures. This is some kind of time-saving
  stuff: students don't spend time copying samples from
  board/projector's screen but receive all samples and diagrams as
  they were shown during lecture. The handouts can be produced from
  presentations.

* Task set for homework. For Computer Science course this might be a
  set of tasks to write different programs.

* A guidance on task solutions. This document might be optional but
  when homework is checked by different teachers it is required that
  every teacher follows the same rules when checking solutions.

* Supplemental materials like coding guidelines, how to interact with
  CI pipeline and so on.

This set of documents has a lot of cross-references and shared
materials:

* The class diagrams, for example, need to included to textbook,
  presentation and corresponding handout. They should be updated in
  all places simultaneously.

* The sample program sources shared between these documents too but
  they places additional requirement: they should be able to run and
  show designed behavior.

* A guidance on task solutions needs to refer various places in
  textbook to connect aspects of task implementation with concepts
  from lectures.

* A textbook needs a bibliography. The textbook might refer specific
  places in other books allowing students to find detailed
  information. This bibliography should be maintained according this
  references.

* A good textbook has index allowing fast search of all concepts end
  entities used in course. For example, index might contain all
  functions, classes, types, constants, modules/header files referred
  in text.

And another aspect should be mentioned explicitly: this set of
documents evolve over the time. The mistakes are fixed, new
information is added, deprecated stuff is removed.

As the result of requirements above a generic office suite can not
handle such document sets effectively. Everything is broken from the
start: to track changes it is wise to place documents under source
control. But these documents represented as binary files and
generating difference between revisions is hard.

Additional tasks for such textbooks are formulas and code
samples. Formatting formulas is hard and various "equation editors"
haven't shown their ability to generate predictable and stable
result. Sample code formatting is very hard in any office suite. The
world wants syntax highlighting because it simplifies reading of the
code. And this highlighting needs to be done manually. Although it is
possible to write such plugin for every office suite I've never seen
such easy-to-use plugin.

It is worth to mention that generic document produced by office suite
breaks some important principles:

* various text parts aligned by inserting white space and
  empty lines very often;

* text formatting (fonts and so on) are applied to text parts
  individually without creation and application of styles. For
  example, mentioning C++ header name in text is done by applying
  monospace font instead of creation of style "C++ Header" and
  application of that style.

## Base Usage Of LaTeX

### Organizing Document

A small document uses `article` class but whole course will be a
`book`. The chapters might contain separate topics. The full content
might be placed to a single file but such huge file is
unmaintainable. The sample code, figures and tables should be shared
between book and presentations so they will live in separate
file. Placing such files in a single directory effectively produces
"entropy warehouse".

The possible solution is splitting all content to files and places
them to separate directories. The corresponding figures, tables and
code samples are placed in the same directory. The top level document
should use `\input` directive to read chapters.

It is convenient have figure and table files in complete form: with
`figure`/`tabular` environments, captions and so on. This gives the
same labels and formatting in all documents.

### Using Semantic Markup

The every entity in document has its own formatting. For example, file
names might have monospace font to differentiate them from
text. Function names often have parenthesises after name in case of C
or C++.

The straight forward approach is using `\texttt` and similar commands
to format them. But this leads to very low level formatting and makes
any changes chard because different entities use the same command.

The proper way is use semantic markup: for every entity should be its
own command to insert it. For example, preamble might contain some
commands like:

```latex
\newcommand{\Filename}{1}{\texttt{#1}}
\newcommand{\Function}{1}{\texttt{#1()}}
```

Please note that `\Function` command is too primitive and will in
improved later.

Such semantic markup not only allows to differentiate formatting for
any entity and process document to extract some data but opens a door
for further improvements.

### Generating Index

The good book has some additional stuff: table of contents, lists of
figures, table and samples, bibliography. All such elements generated
by LaTeX automatically if proper commands used. The index always
requires more work.

For example, we want to have index items for:

* all C and C++ headers;
* all functions;
* all classes;
* all global constants
* an so on.

To generate index the following steps required:

1. Add package `makeidx`
2. Insert `\makeindex` in preamble.
3. Insert command `\index` in appropriate places.
4. Insert index at final part of document:

   ```latex
   \phantomsection
   \addcontentsline{toc}{chapter}{\indexname}
   \printindex
   ```

   The `\phantomsection` generates proper page number when added to
   table of content with `\addtocontentsline`.
5. Process extracted index information with `makeindex` command.
6. Make additional run of LaTeX to include generated index.

The steps 1, 2 and 4 are performed once, the steps 5 and 6 are easily
automated via scripts and/or Make. But step 3 is manual.

Using semantic markup it is possible to automate this task. For
example, any function from public API should be added to index twice:

* first time under its name and
* second time as subelement of "function".

To do this the `\Function` command can be extended:

```latex
\newcommand{\Function}{1}{\texttt{#1()}%
\index{#1@\texttt{#1}}\index{function!#1@\texttt{#1}}}
\WithSuffix\newcommand{\Function*}{1}{\texttt{#1()}}
```

To sort properly and keep formatting the `@` symbol is used in index
entry: left side is used to sort and right side to show name. The
command `\Function*` is added to insert function name without updating
index. This might be required to refer to private sample function.

Performing such task for every semantic object builds correct index.

A note about file names: the C and C++ uses files as
headers. But header is a separate concept so it would be nice to have
separate commands `\Filename` and `\CHeader`.

A note about constants: the constants can be very different. For
example, they can be an error code, a POSIX signal identifier or just
a value. So the nice index places constants to corresponding
category. To achieve this it the command for constant can accept 2
parameters:

1. an optional category with default empty value and
2. constant name.

If category is not empty constant added to index under specified
category:

```latex
\newcommand\CxxSymbol[2][]{\texttt{#2}}\index{#2@\texttt{#2}}
\ifx\relax#1\relax\else\index{#1!\texttt{#2}}\fi}}
\WithSuffix\newcommand\CxxSymbol*[1]{\texttt{#1}}
```

Two versions created again: one to add value to index and starred to
skip index update.

The C and C++ constants contain underscores very often. To avoid
broken format additional function can be used to replace them in
formatting part of commands.

### A Special Case: UML

Many Computer Science related documents use UML diagrams as standard
way to show relations between entities. Although it is possible to
draw them in UML editor and export as picture to include to LaTeX
document this approach hits some limitations:

1. Raster images huge and scales badly.
2. XeLaTeX accepts only PDF as vector image.

But LaTeX can draw diagrams for you. The library
[TikZ](https://tikz.dev/) is very good in drawing pictures. All that
left is to make special UML commands to draw entities and relations
between them.

The nice library
[TikZ-UML](https://perso.ensta-paris.fr/~kielbasi/tikzuml/) done this
work almost ideally. The following issues exist:

* Relation stereotypes can be drawn over relation making them hard to
  read.
* Some issues with sloped relation stereotypes (attribute is not
  supported).
* Beamer overlays might work incorrectly (see below).

## Presentations and Handouts

The `beamer` document class allows to make nice presentations in
LaTeX. Powered by packages it adds some facilities to make life
easier:

* theme support,
* overlays allowing progressive slide creation,
* notes for presenter,
* document modes
* and so on.

Some amount of commands and styles are overridden to suite
presentation requirements, for example, bibliography has style
features.

The Beamer allows to write document and render it in different modes:
*presentation*, *article* and *handout*. Personally I don't use
*article* mode because I want to have a book-like document.

### Themes

The Beamer comes with theme support and extensive theme list. Themes
allow to change every single aspect of presentation, for example, how
to titles shown, where to put logo and so on.

The default themes is good enough but your company has its own brand
book dictating how all such things must look. So you apply brand book
in 3 ways:

* creating a new [class](#classes-and-packages) on top of `beamer`,
* creating a new [package](#classes-and-packages) or
* creating a new theme.

Without diving to details the proper way is creating a theme. The
allows simple switch of presentation style by using command
`\usetheme{THEME}` in the preamble. When you work on presentation with
people outside your organization who lacks access to fonts, icons and
so on this gives ability to use optional theme:

```latex
\IfFileExists{beamerthemeAcme.sty}{\usetheme{Acme}}{\usetheme{DEFAULT-LOOKS-LIKE}}
```

The front and last page of corporate presentations are complex very
often. The use background images and specially placed fields. These
pages can be created as [TikZ](https://tikz.dev/) picture which allows
precise placement of nodes with text. The following aspects should be
took in consideration:

* The Beamer frame sizes and they don't match corporate styles. All
  placements, font sizes and scales should be transferred from
  corporate coordinates to Beamer coordinates. It is possible to
  perform this task automatically via Pgf but if corporate style
  doesn't change very often it is much easier to use calculator.

* The baseline skip for fonts should be carefully calculated because
  simple rules might not work with designer's vision.

* The background images is another pain point: they are raster very
  often. But since they need to scale designers provide them in
  gigantic resolutions usually. Such images take space in resulting
  PDF and slows down rendering. Having vector background is very nice
  but usually background is result of complex blending, shadows,
  gradients and so on and as the result vector image is not available.

* The extra attention should payed to versions of background
  images. The first and the last slide might differ in tiny
  details. The backgrounds may evolve over the time.

### Presentation Titles and Textbook

When a set of textbook and presentations is prepared the presentation
title might follow chapter title possibly including chapter
number. For example, the tenth chapter of C++ course might be named
"Resource Management" and its presentation title should be
"10. Resource Management".

This can be achieved automatically by adding small piece of code and
tiny addition to the [document organization](#organizing-document):

* Each chapter not only placed to its own directory but have fixed
  file name, for example, `content.tex`.

* Chapter file contains only content and doesn't include command
  `\chapter` with title.

* The top-level document uses special command to add chapter, include
  file and write some commands to additional file remembering chapter
  index and title.

* Presentation includes file generated in previous step and uses
  commands from it to refer chapter information.

The straightforward implementation might look like (place it into
preamble):

```latex
\newwrite\ChapterFile
\openout\ChapterFile=\jobname.chap

\newcommand\InsertChapter[3]{
  \chapter{#2}

  \label{chap:#1}

  \input{#3/content}

  \immediate\write\ChapterFile{
    \string\newcommand\string\ChapterIndex#1{\thechapter}
  }
  \immediate\write\ChapterFile{
    \string\newcommand\string\ChapterName#1{\unexpanded{#2}}
  }
  \immediate\write\ChapterFile{
    \string\newcommand\string\ChapterTitle#1{\thechapter. \unexpanded{#2}}
  }
}
```

The generated file will be placed near output file, will have the same
name but suffix `.chap`. Every chapter has unique identifier provided
as the first parameter of `\InsertChapter` and this identifier
appended to generate commands to refer chapter data. Chapter title
goes in the second parameter and subdirectory in the third. So, if
textbook includes command

```latex
\InsertChapter{ResourceManagement}{Resource Management}{resource-mangement}
```

The book will be compiled with (for convenience a label for chapter is
added automatically to make cross-references in the text book)

```latex
\chapter{Resource Management}
\label{chap:ResourceManagement}
\input{resource-management/content}
```

The chapter file will have the following commands defined (assuming
that this is 10th chapter):

```latex
 \newcommand\ChapterIndexResourceManagement{10}
 \newcommand\ChapterNameResourceManagement{Resource Management}
 \newcommand\ChapterTitleResourceManagement{10. Resource Management}
```

And presentation can define its title like:

```latex
\title{\ChapterTitleResourceManagement}
```

### Reusing Figures and Tables

As mentioned [above](#organizing-document) the figures and tables
should be included from separated files as is with captions, labels
and so on. This leads to the following issues:

* Captions aren't needed in presentations in general case. So they can
  be switched off via setting empty Beamer templates:

  ```latex
  \setbeamertemplate{caption}{}
  \setbeamertemplate{caption label separator}{}
  ```

* Figures and tables might not fit the slide. This requires careful
  formatting to achieve proper look in textbook and presentation.

* When figures and tables in presentation should be shown
  progressively (Beamer has word "overlay" for this) this feature
  collides with textbook: to construct overlays Beamer uses "overlay
  spec" (in angle brackets) and special commands like `\pause`,
  `\only<>` and so on. To handle this in texbook such commands should
  be defined as doing nothing. This effectively produces final
  figure. But overlay spec is much harder to handle so it just can be
  avoided.

### Handouts

The biggest underestimated feature of presentation programs is Presenter
Notes. The presentation is not a textbook and is not an
article. Placing too much text to it makes a "slidument" instead of
presentation. A good presentation only has illustrative material and
absolutely required definitions because presentation is created to
assist speaker.

In such case it is relatively easy to forget to speak about some
important aspects and corner cases especially when talking about
presentation's topic once a year. Presenter notes allow to inform
speaker about such things. The PowerPoint has this, Apple Keynote has
this and Beamer offer's this.

The notes are inserted to slide via `\note{text}` command. To turn on
presenter notes the commands

```latex
\setbeameroption{show notes}
\setbeameroption{show notes on second screen}
```

should be added to preamble. This by default render every page split
in two parts:

1. Left is a slide.
2. Right is a presenter screen, showing note, section, title, subtitle
   and miniature of slide.

The special program like
[SplitShow](https://github.com/mpflanzer/splitshow/) can split such
page and show slide on one monitor and presenter screen on another.

Some samples are to complex to copy their contents from projected
presentation: this takes time and introduces mistakes. Tables and
diagrams in the same category because reproducing them in student's
notes takes time. The Beamer offers a solution for this: generating
handouts. When option `handout` is passed to `beamer` class output
document rendered in special mode `handout`:

* simple overlays (without mode specification) rendered in final
  state,
* notes are removed.

The resulting PDF can be given to students to reduce amount of their
work.

## Integrating Sample Code

The almost any Computer Science textbook will contain a lot of code
samples. To have a good course these samples require a lot of
attention.

At the very first they should be free of errors in ideal case except
for samples that show errors and mistakes. So every sample should be
in a complete form suitable to be compiled (for languages with
compiler) and run.

The textbook might include whole samples but for presentation it's
impossible:

* every slide allows only small amount of information,

* sample contains significant part which is required for sample to run
  but goes beyond the scope of particular slide.

To solve this issue the `listings` package gives control over what is
inserted. The command `\lstinputlisting` accepts file name as required
parameter and list of options as optional parameter. The most
interesting are:

* `label`: adds label to refer to the listing.

* `caption`: adds caption to listing, usually not required in
  presentation but necessary in textbook.

* `linerange`: defines range of lines to include to listing, can be
  given as line numbers or named range markers.

* `firstnumber`: number of first line in the listing, can be a number
  or word `last` to continue numbering from previous listing.

  > ATTENTION: sometimes `firstnumber` should have value of correct
  > number - 1.

Usage of line numbers as range definition is fragile because sample
can be updated and line numbers might change. After that textbook and
presentation should be change to refer to new line numbers.

The range markers are less fragile but require additional setup. Range
markers should be inserted to source of the sample and must not affect
its behavior so they go to the comments. The correct delimiter for
range markers should be defined for every listing via 2 parameters
`rangeprefix` and `rangesuffix`. For example, C and C++ can define
them as

```
\lstinputlisting[rangeprefix=/*\{,rangesuffix=\}*/,includerangemarker=false...]{...}
```

So the C code can include something like

```c++
...

/*{mainBegin}*/
int main()
{
  ...
}
/*{mainEnd}*/
```

To insert `main()` contents the `linerange={mainBegin-mainEnd}` should
be added to listing.

Typing range prefix and suffix for every listing is tiresome and error
prone. When all samples are written in single language they can be
defined via `\lstset`. More generic solution is to define special
command which set them automatically. For example,

```latex
\newcommand{\cxxinput}[2][]{\lstinputlisting[rangeprefix=/*\{,rangesuffix=\}*/,includerangemarker=false,#1]{#2}}

```

To refer sample's lines in text labels should be defined. To do this
the `\label` command should be inserted to sample code. The Listings
package looks for prefix and suffix defined via option `escapeinside`
to detect and execute arbitrary LaTeX code. For C and C++ it might
be defined as `escapeinside={/*$}{$*/}` so listing might look like

```c++
/*{mainBegin}*/
int main()
{
  ...
  return 0; /*$\label{return-exit-code}$*/
}
/*{mainEnd}*/
```

Additional small notes about listings in presentations:

* The listings (and any verbatim environment) can easily break
  compilation of Beamer frame. In this case frame should have option
  `fragile`.

* The listings usually follow coding guidelines. But these guidelines
  might lead to horizontal and vertical overfows. The first step to
  solve these issues is trying to reformat code with minor guidelines
  violations. For example, remove blank lines, collect many variables
  with the same type on single lines and so on.

* The second step to solve issue with long listing is trying to split
  it to several listings and place them to consecutive slides or to
  columns.

* As the final resort the option `shrink=<factor>` can be added to
  frame to reduce font size. But this can lead to unreadable slide.

* Progressive (overlay in Beamer terms) listings can be created by
  using `escapeinside` and `\pause` command. But if listing has
  non-transparent background to create a visual block this block will
  grow from frame to frame along with uncovering additional lines. An
  extra markup required to get full size block from the beginning.

* Sometime listing in presentation omits some irrelevant fragment. The
  ellipsis can be inserted via `escapeinside` and `\ldots`. The small
  bonus: this ellipsis visually differs from three points used as
  syntactic construction in C and C++.

## Version Control

The whole materials (texbook, presentations and so on) evolves over
the time the users should have ability to track changes. The simplest
way is to put all sources under version control system like
[Git](https://git-scm.org).

This allow:

* collaborative work via branches and Pull/Merge Requests,

* tracking what and when changed,

* tagging explicit versions.

To get versions from version control system and put them to document
additional effort required. The Git allows to query currently checked
out branch via `git describe --dirty='*' --first-parent`. The
`--dirty` flag tell Git which symbol should be put to output when
local repository has uncommitted changes and the `--first-parent`
forces traversal by current branch ignoring merges. The output will
have name of latest tag with number of commits since it and `*` in
case of uncommitted changes. If no tag present the output will be
empty and it is possible to fall back to `git rev-parse --short HEAD`
to get short hash of commit as version.

The resulting string needs to be fed to LaTeX. The easiest way is
changing LaTeX command from

```sh
$ pdflatex source.tex
```

to

```sh
$ pdflatex "\\def\\Version{$VERSION}\\input{source.tex}"
```

The command `\Version` can be used in document to insert version
number text.

## Making Stuff Reusable

All this stuff should be placed somewhere. When a single and unique
document created the possible place is preamble of the document. When
set of documents created this approach ineffective.

### Classes And Packages

The proper solution is create custom packages and classes. The Beamer
themes are packages by its nature with special prefix `beamertheme` in
name.

All commands and styles might be split between packages in the
following way:

* All Beamer-related stuff (formatting titles, title/final pages and
  so on) should go to Beamer theme. It can be split to outer, inner,
  color and font themes if needed.

* Color definitions and additional commands and settings required for
  shared figures, table should go to a separate package to be
  available in textbook an presentation. This package should be loaded
  by Beamer theme automatically.

* If set of documents use the same set of packages and settings it is
  possible to reduce code duplication by defining a custom class. But
  defining custom class might make integration with other classes
  hard. I would suggest to create class as the very last resort. The
  package might be created instead.

### A Word About Fonts

The company style guide often refers to custom fonts. Historically
LaTeX handles fonts via METAFONT system. Fonts from METAFONT give nice
results but most of modern fonts are in OpenType, TrueType or
Adobe Type1 formats. The classic PDF LaTeX unable to handles these
types.

To use such fonts switch to XeLaTeX required. XeLaTeX has native
support of OpenType and TrueType fonts. Another nice feature is
Unicode handling without magic transcoding.

The XeLaTeX can use fonts installed on system by their names or
directly from files.

The first approach is good enough for one time documents. If document
is maintained by many people on different systems the maintaining up
to date font set becomes hard.

The better way is placing fonts into version control along with
packages from previous section and referring them by file names.

To use fonts in document the `fontspec` package should be loaded. For
example, [DejaVu](https://dejavu-fonts.github.io) font family can be
registered like

```latex
\setmainfont{DejaVuSerif}[
  Ligatures      = TeX,
  Path           = fonts/,
  Extension      = .ttf,
  BoldFont       = *-Bold,
  ItalicFont     = *-Italic,
  BoldItalicFont = *-BoldItalic,
  SmallCapsFont  = DejaVuSerifCondensed-Bold,
]
\setsansfont{DejaVuSansCondensed}[
  Ligatures      = TeX,
  Path           = fonts/,
  Extension      = .ttf,
  BoldFont       = *-Bold,
  ItalicFont     = *-Oblique,
  BoldItalicFont = *-BoldOblique,
]
\setmonofont{DejaVuSansMono}[
  Path           = fonts/,
  Extension      = .ttf,
  BoldFont       = *-Bold,
  ItalicFont     = *-Oblique,
  BoldItalicFont = *-BoldOblique,
]
```

Please note that switching fonts might not be so easy. Visual size of
letters of different fonts can differ even in the same family (for
example, Liberation family monospace font looks higher than serif) and
different metrics can lead to another overflows and underflows.

### TeX Directory Structure

The simple way to places all classes, packages and other assets like
fonts and images is creation of subdirectory, for example, `tex` and
place all stuff there. The path to directory must be added to
environment variable `TEXINPUTS` to make them available to XeLaTeX.

But this simple way is not convenient when all this facilities need to
be reused in various unrelated documents. In such case a more complex
approach helps to solve the issue.

The [TeX Directory Structure](https://tug.org/tds/tds.html) (TDS)
describes how to place assets to allow TeX implementation to find
them. Long story short the packages and classes go to
`tex/latex/<subdir>` directory, fonts to `fonts/<type>/<subdir>`
directory and images to `tex/generic/images/<subdir>` directory.

When files placed according this guideline it is relatively easy to
add them to TeX implementation.

#### Integration with TeXLive/MacTex

There are 2 ways to add repository with TDS structure to TeXLive:

1. Copy (or link) files to a directory configured in `TEXMFHOME`
   configuration parameter. To get this parameter the following
   command can be used:

   ```sh
   $ kpsewhich --var-value=TEXMFHOME
   /home/user/texmf
   ```

2. Override `TEXMFHOME` environment variable. The new value can be
   written to distribution configuration file or set in the
   environment. Please note that if multiple directories need to be
   configured the syntax with braces should be used:
   `{dir1:dir2:dir3}`.

#### Integration with MikTeX

The `miktex-console` tool handles configuration information. The path
to repository should be added to "TEXMF root directories" list on
"Directories" page in section "Settings".

## Conclusion

The great LaTeX system allows to make perfect documents and
presentations. But creation of maintainable set of documents requires
and extra attention and some diligence. Personally I've found that
recommendations above dramatically simplifies this work.

## Changelog

29 August 2024
: Initial version.

