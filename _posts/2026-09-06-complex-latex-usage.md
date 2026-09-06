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
XeLaTeX unless compatibility issues explicitly stated.

> NOTE: this article will update with new information and fixes. Its
> publishing date will change accordingly.

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

Lets take as example an university course. The good course might
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
  [CI](https://en.wikipedia.org/wiki/Continuous_integration) pipeline
  and so on.

This set of documents has a lot of cross-references and shared
materials:

* The class diagrams, for example, should be to be included to the
  textbook, presentation and corresponding handout. They should be
  updated in all these places simultaneously.

* The sample program sources are shared between these documents too
  but they place additional requirement: they should be able to run
  and show required behavior.

* The guidance on task solutions needs to refer various places in
  textbook to connect aspects of the task implementation with concepts
  from lectures.

* The textbook needs a bibliography. The textbook might refer specific
  places in other books allowing students to find detailed
  information. This bibliography should be maintained according this
  references.

* A good textbook has index allowing fast search of all concepts and
  entities used in the course. For example, index might contain all
  functions, classes, types, constants, modules/header files referred
  in the text.

So these set of documents gives the following process:

![LaTeX Processing](/assets/images/latex-process.svg)

Another aspect should be mentioned explicitly: this set of
documents evolve over the time. The mistakes are fixed, new
information is added, deprecated stuff is removed.

As the result of the requirements above a generic office suite can not
handle such document sets effectively. Everything is broken from the
start: to track changes it is wise to place documents under source
control but these documents represented as binary files and
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

Please note that this doesn't mean "every document produced by office
suite". It is _possible_ to create styles in document and apply them
but in my experience most of such documents don't have them.

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

It is convenient to have figure and table files in a complete form: with
`figure`/`tabular` environments, captions and so on. This gives the
same labels and formatting in all documents.

### Using Semantic Markup

The every entity in document has its own formatting. For example, file
names might have monospace font to differentiate them from
text. Function names often have parenthesises after name in case of C
or C++.

The straightforward approach is using `\texttt` and similar commands
to format them. But this leads to very low level formatting and makes
any changes hard because different entities use the same command.

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
   table of contents with `\addtocontentsline`.
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

To sort item properly and keep formatting the `@` symbol is used in
index entry: the left side is used to sort and the right side to show
name. The command `\Function*` is added to insert function name
without updating the index. This might be required to refer to a
private sample function.

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

Two versions created again: one to add value to the index and starred
version to skip the index update.

The C and C++ constants contain underscores very often. To avoid
broken formatting in resulting document an additional function can be
used to replace underscores in the formatting part of commands.

### A Special Case: UML

Many Computer Science related documents use UML diagrams as standard
way to show relations between entities. Although it is possible to
draw them in UML editor and export as picture to include to LaTeX
document this approach hits some limitations:

1. Raster images huge and scales badly.
2. XeLaTeX accepts only PDF as vector image.

But LaTeX can draw diagrams for you. The library
[TikZ](https://tikz.dev/) is very good in drawing pictures. All that
left to author is to make special UML commands to draw entities and
relations between them.

The nice library
[TikZ-UML](https://perso.ensta-paris.fr/~kielbasi/tikzuml/) done this
work almost ideally. The following issues exist:

* Relation stereotypes can be drawn over relation making them hard to
  read.
* Some issues with sloped relation stereotypes (attribute is not
  supported).
* Beamer overlays might work incorrectly (see below).
* Be careful with node identifiers. Although they aren't shown they
  can contribute to node size: the long identifier for node with
  short name can create huge box. The `\umlbasicstate` is known command
  with this behavior.

## Presentations And Handouts

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
*article* mode because I want to have a book-like document and it is
much easier to create in with `book` class.

### Themes

The Beamer comes with theme support and extensive theme list. Themes
allow to change every single aspect of presentation, for example, how
to titles shown, where to put logo and so on.

The default themes are good enough but your company has its own brand
book dictating how all such things must look. So author can apply
brand book in 3 ways:

1. by creating a new [class](#classes-and-packages) on top of `beamer`,
2. by creating a new [package](#classes-and-packages) or
3. by creating a new theme.

Without diving to details the proper way is creating a theme. The theme
allows simple switch of presentation style by using command
`\usetheme{THEME}` in the preamble. When you work on presentation with
people outside your organization who lacks access to fonts, icons and
so on this gives ability to use an optional theme:

```latex
\IfFileExists{beamerthemeAcme.sty}{\usetheme{Acme}}{\usetheme{DEFAULT-LOOKS-LIKE}}
```

The front and the last page of corporate presentations are complex by
their nature. They use background images and specially placed
fields. These pages can be created as [TikZ](https://tikz.dev/)
picture which allows precise placement of nodes with text. The
following aspects should be took in consideration:

* The Beamer frame sizes might not match corporate style's frame
  size. All placements, font sizes and scales should be transferred
  from corporate coordinates to Beamer coordinates. It is possible to
  perform this task automatically via Pgf but in case when corporate
  style doesn't change very often it is much easier to use calculator.

* The baseline skip for fonts should be carefully calculated because
  simple rules might not work with designer's vision.

* The background images is another pain point: they are raster very
  often. But since they need to scale designers provide them in
  gigantic resolutions usually. Such images take space in resulting
  PDF and slows down rendering. Having vector background is very nice
  but usually background is result of complex blending, shadows,
  gradients and so on and as the result vector image is not available.

* The extra attention should be paid to versions of background
  images. The first and the last slide might differ in tiny
  details. The backgrounds may evolve over the time.

### Customizing Elements

In case of corporate presentation the brand book dictates how various
stuff should look. This includes [fonts](#a-word-about-fonts), colors,
various graphics including logo. The title frame almost always doesn't
match any existing theme because designer tries to make it
recognizable. As the result presentation requires additional efforts.

Lets take title page as example of possible ways of
customization. When title page needs to be changed the following
options exists:

1. Just make regular frame and insert content inside. Taking this idea
   up to limit gives "just insert picture taken from Power
   Point". The picture version is ineffective because it doesn't use
   standard attributes of document: title, author names, date and so
   on. In general this works when only 1 presentation is
   required. Making set of presentations requires another solution.

2. Redefine `\maketitle` to provide new content. This allows title
   page to be reused as part of Beamer theme. But overriding
   `\maketitle` requires duplication of its behavior: this command can
   be used inside frame or outside frame. In latter case it adds frame
   automatically. This behavior is given by Beamer.

3. Redefine `\titlepage`. This command is used by `\maketitle` to
   create contents of title frame. This is much easier than redefining
   `\maketitle` since `\titlepage` can be used only inside frame.

4. The solutions above ignore Beamer way to customize things. To
   utilize Beamer facilities the template `title page` should be
   updated.

To change Beamer template the following ways available (please look
more details of commands below in the Beamer documentation):

1. `\setbeamertemplate{title page}{<content>}` changes title page to
   `<content>`.

2. `\defbeamertemplate*{title page}{<name>}{<content>}` makes named
   (with `<name>`) template and sets is as current. This allows to
   quick switch template with short command. For example,
   `\setbeamertemplate{title page}[default]` selects default template
   and `\setbeamertemplate{title page}[<name>]` selects the new
   one. It is possible to pass additional parameters to template so
   look for details in documentation.

   Although it looks like overkill for title frame for other aspects
   this might be more efficient. For example, templates for itemizing
   elements can be switched easily.

   In case when main title frame design leads to ugly word wrapping,
   for example, in presentation's title designers can offer "back up
   design" with another layout. In this case using
   `\defbeamertemplate` allows to define both cases, select main
   design as default and give ability to switch title page design to
   back up version in selected presentations.

The simple title page can be created using `\vskip` and
`beamercolorbox` but corporate design might require exact positioning
over background image. In this case TikZ picture can be created with
precise text node placement. Please note that this will require

```latex
\begin{tikzpicture}[remember picture,overlay]
  …
\end{tikzpictire}
```

to achieve proper placement and at least 2 LaTeX runs to get final result.

During template creation other templates should be used as much as
possible because it gives ability to tune various things up. To select
styles from template `\usebeamerfont` and
`\usebeamercolor` insert required formatting. The `\usebeamertemplate`
inserts template text. In case of TikZ-based title page applying
templates for title, author, date and so on might be tricky so they
can be avoided by placing text directly to node.

### Presentation Titles And Textbook

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

The book will be compiled with (for convenience a `\label` for the
chapter is added automatically to make cross-references in the text
book):

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

It is possible to go further. The current solution makes automation
around title frames hard. For example, it is convenient to have
*current* chapter concept. To achieve this additional command is
required:

```latex
\ProvideDocumentCommand{\SelectChapter}{m}{
  \def\CurrentChapterName{\csname ChapterName#1\endcsname}
  \def\CurrentChapterIndex{\csname ChapterIndex#1\endcsname}
  \def\CurrentChapterTitle{\csname ChapterTitle#1\endcsname}
}
```

> NOTE: `\SelectChapter` is implemented with new facilities for
> commands: `NewDocumentCommand`, `RenewDocumentCommand`,
> `ProvideDocumentCommand`  and `DeclareDocumentCommand` available in
> modern LaTeX2e implementations. For more details take a look at
> [documentation](https://texdoc.org/serve/usrguide/0).

The `\SelectChapter{<id>}` makes chapter with `id` "current" by
setting 3 macros. According to example above, the call to
`\SelectChapter{resource-management}` will set `\CurrentChapterName`
to "Resource Management", the `\CurrentChapterIndex` to 10 and the
`\CurrentChapterTitle` to `10. Resource Management`. This allows to
refer to the current chapter without specific knowledge about
identifier reducing amount of information to pass around.

### Reusing Figures And Tables

As mentioned [above](#organizing-document) the figures and tables
should be included from separate files as is with captions, labels
and so on. This leads to the following issues:

* Captions aren't needed in presentations in general case. So they can
  be switched off via setting empty Beamer templates:

  ```latex
  \setbeamertemplate{caption}{}
  \setbeamertemplate{caption label separator}{}
  ```

* Figures and tables might not fit the slide. This requires careful
  formatting to achieve proper look in the textbook and the
  presentation.

* When figures and tables inserted into the presentation need to be
  be shown progressively (Beamer has word "overlay" for this) this
  feature collides with textbook: to construct overlays Beamer uses
  "overlay spec" (in angle brackets) and special commands like
  `\pause`, `\only<>` and so on. To handle this in textbook such
  commands should be defined as doing nothing. This effectively
  produces final figure. But overlay spec is much harder to handle so
  it just can be avoided.

#### Generating SVG

The presentations, articles and books are done in PDF format. This
gives exactly same representation on any device. But sometimes figures
should be included into Web document. Browsers may render PNG, JPEG or
SVG inline. PNG and JPEG are raster formats so they scale badly. SVG
is vector format and TikZ output can be done in SVG with small
efforts. The process is documented in [PGF/TikZ
Manual](https://tikz.dev/drivers#sec-10.2.4) and below is short
excerpt:

1. Create a minimal document with `dvisvgm` driver and save it as
   `figure.tex`:

   ```latex
   \documentclass[dvisvgm]{minimal}

   \usepackage{tikz}

   \begin{document}
   \end{document}
   ```

2. Insert figure definition into `document` environment.

3. Generate DVI:

   ```
   latex figure.tex
   ```

4. Generate SVG:

   ```
   dvisvgm figure.tex
   ```

Please note that the resulting `figure.svg` might be different from
PDF version. I found that:

1. Some letters clash in SVG. Usually this means that single string is
   split to two or more `tspan`s with shifted positions. This fix
   this just remove extra `tspan`s from document.

2. Some nodes might be positioned differently. This can be adjusted
   manually or by vector graphics editor.

Regardless of issues above this process saves time from repeating
figure in vector graphics editor. The picture in this article is
generated according to steps above.

### Handouts

The biggest underestimated feature of presentation programs is Presenter
Notes. The presentation is not a textbook and not an article. Placing
too much text to it makes a "slidument" instead of presentation. A
good presentation only has illustrative material and absolutely
required definitions because presentation is created to assist speaker.

In such case it is relatively easy to forget to talk about some
important aspects and corner cases especially when talking about
presentation's topic once a year. Presenter notes allow to inform
speaker about such things. The PowerPoint has this, Apple Keynote has
this and Beamer offers this.

The notes are inserted to slide via `\note{text}` command. To turn on
presenter notes the commands

```latex
\setbeameroption{show notes on second screen}
```

should be added to preamble. This by default render every page split
in two parts:

1. Left is a slide.
2. Right is a presenter screen showing note, section, title, subtitle
   and miniature of slide.

The special program like
[SplitShow](https://github.com/mpflanzer/splitshow/), [Dual-Screen
PDF Viewer](https://dspdfviewer.danny-edel.de) or
[pdfpc](https://pdfpc.github.io/) can split such page and show slide
on one monitor and presenter screen on another.

> NOTE: The XeTeX has issue with text on slides with notes: the
> foreground color of the content is matched with background color. To
> fix this the simplest way is resetting font at the beginning of
> every frame. This can by achieved by inserting to preamble:
> ```latex
> \makeatletter
> \def\beamer@framenotesbegin{\usebeamercolor[fg]{normal text}%
>   \gdef\beamer@noteitems{}%
>   \gdef\beamer@notes{}%
> }
> \makeatother
> ```

Some samples are too complex to copy their contents from projected
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

Since `handout` is a class option this requires to update source file
to have this option included into `\documentclass` statement. Please
take a look at [Passing Document Class
Options](#passing-document-class-options) on how to avoid changing
source files.

### Issues With overlays

The important difference between `\only` and `\onslide` greatly
affects presentation: the content of `\only` doesn't occupy space on
frames not matched overlay specification but content of `\onslide`
does. This allows to create stack of pictures, for example, when every
level of stack has its own frame. In book all these pictures will be
placed one under/aside other. This may lead to vertical or horizontal
overflow and clipping.

The better result can be achieved by:

* splitting big content to parts;
* placing every part to individual presentation frame manually;
* collecting parts to single figure in book.

### Bibliographies

In any university course bibliography is very important part: it gives
source for additional information. Creating bibliography in book or
article is simple and straightforward:

1. Information about sources is collected in `bib` files. Some
   information can be found in Internet, for example, ANSI and ISO
   standards collections.

2. The sources are referenced via `\cite{item}` or
   `\cite[text]{item}`. The former inserts reference to source and the
   latter allows to refer exact element of source (table, figure,
   source code or page).

3. The resulting bibliography is inserted via `\printbibliography`

> NOTE: do not forget that bibliography requires additional
> translation to get all references right. This behavior is same as
> index.

In presentations this a little bit tricky. Since reference to source
is not required on visible frame the citation might be placed to
notes. Unfortunately this immediately makes empty bibliography in
handout mode since notes are removed.

The solution is placing `\nocite{item}` command on visible
frame. Output of this command doesn't occupy space since it completely
empty. The source item is properly referenced and inserted to
bibliography. Such `\nocite` commands might be placed to title frame
referencing all required items or in case of frequently changing
presentation to corresponding frames. It is still possible to insert
`\cite` to notes to have speaker visible references to books if this
required.

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

* Sometimes listing in presentation omits some irrelevant fragment. The
  ellipsis can be inserted via `escapeinside` and `\ldots`. The small
  bonus: this ellipsis visually differs from three points used as
  syntactic construction in C and C++.

### Existing Issues

It is possible to insert more than one range of source code to
single listing. Unfortunately this arises some issues:

* Inserting code with range markers inside like
  `linerange={mainBegin-mainEnd}` with source


   ```c++
/*{mainBegin}*/
int main()
{
  ...
  /*{returnBegin}*/
  return 0; /*$\label{return-exit-code}$*/
  /*{returnEnd}*/
}
/*{mainEnd}*/
   ```

  will show intermediate markers inside. This is inconvenient in most
  cases. It is possible to hide them by assigning font color matched
  to background color but this doesn't prevent copying of them.

* Inserting multiple ranges via
  `linerange={mainBegin-mainEnd1,returnBegin-returnEnd}` is possible
  only when `mainEnd1` and `returnBegin` are placed on different
  lines.

* When multiple consecutive ranges are inserted in the same listing
  and line numbers are got from source file the hole in numbering
  exists because listing misses number from range marked line.

* In case of assigned first line number to listing like
  'firstnumber=1` all ranges in listing will start from this number
  which is inconvenient.

## Version Control

When the whole materials (texbook, presentations and so on) evolve over
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

All these stuff should be placed somewhere. When a single and unique
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

### Parameters Of Custom Classes

The custom class usually will rely on some other class because this
dramatically reduces efforts on writing. So it will load other class
via `\LoadClass` command. For example:

```latex
\NeedsTeXFormat{LaTeX2e}[2023-11-01[
\ProvidesClass{augmented-book}[2026-09-06 Augmented book]

% Preparations

\LoadClass{book}

% More definitions
```

Such definition creates a class without any parameter. To change its
behavior the parameter should be added. The modern way is usage of
`\DeclareKeys`. The complete reference can be found in LaTeX
collection of documentation. The most interesting aspect is how to
make created class transparent on underlying class options.

Since it is hard and error prone to duplicate all parameter
definitions from underlying class the new class should pass unknown
parameters to underlying class. This can be achieved with the
following code:

```latex
\DeclareKeys[AugmentedBook]{
  parameter.if = \if@AB@parameter,
}
\DeclareUnknownKeyHandler[AugmentedBook]{
  \PassOptionsToClass{\CurrentOption}{book}
}
\ProcessKeyOptions[AugmentedBook]

\LoadClass[10pt[{book}
```

The code above declares a parameter `parameter` for `augmented-book`
class. Its declaration creates a switch which can be used like `\if`
without condition.

The `\DeclareUnknownKeyHandler` just pass current option value as is
to `book` class.

The `\ProcessKeyOptions` triggers processing.

So when LaTeX processes
`\documentclass[parameter,a4paper]{augmented-book}` it does:

1. The switch `\if@AB@parameter` is set to execute `true` path because
   `parameter` is set.
2. The `a4paper` is not known by this package so handler from
   `\DeclareUnknownKeyHandler` is triggered and `\CurrentOption` is
   set to `a4paper`.
3. The `\PassOptionsToClass` sets `a4paper` as one of the parameters
   to pass to `book` class.
4. The `\LoadClass{book}` loads class `book` and passes `10pt` from
   `\LoadClass` and `a4paper` from `\PassOptionsToClass`.

The same technique can be used for packages by using
`\PassOptionsToPackage`. More interesting that parameters from
`\PassOptionsToPackage` will be passed regardless of which command
loads package: `\RequirePackage` or `\usepackage`.

### Passing Document Class Options

Passing "temporary" options to document class is a task which requires
some additional efforts because LaTeX commands (`latex`, `pdflatex`,
`xelatex` etc) don't have any switch to pass extra option to document
class or define any symbol. To get this done additional facility needs
to be handcrafted.

The LaTeX commands can accept as parameter not only file name but
LaTeX command. So the following 2 invocations are completely the same:

```
latex source.tex
```

```
latex '\input{source}'
```

This gives some flexibility because it is possible to use some LaTeX
commands to alter behavior inside `source.tex`.

The following options are available:

1. Define a variable and use it inside.

   For example, putting `\def\Handout{}\input{presentation}` allows to
   write in the `presentation.tex`:

   ```latex
\documentclass[\Handout]{beamer}
```

   This effectively passes empty options to Beamer. But when build
   system builds handouts it can rewrite command like
   `\def\Handout{handout}\input{presentation}` giving instruction to
   Beamer.

   Although this is working and recommended scenario it has some
   drawbacks when more than one option needs to be passed it this way
   or another options already set in source file:

   * The comma (`,`) sometimes needs to be added to `\def` even when
     option is not passed.
   * Empty (not passed)options sometimes might be handled as unknown
     option. If target class is strict on its option this breaks
     compilation.

   The benefit of this solution that the defined value can be used on
   any level including source file but in general it is a bad practice
   because such depedencies in sources are extremely hard to trace.

2. Request passing option directly to class with
   `\PassOptionsToClass`.

   For example, the
   `\PassOptionsToClass{handout}{beamer}\input{presentation}` does the
   same sample before.

   At first glance the drawback of this solution is required knowledge
   of the used document class in `presentation.tex`. But should not be
   a problem:

   * The class usually is known by nature of document and build
     receipt.

   * If document uses custom class which eventually loads specified
     class via `\LoadClass` the option will be passed to it anyway
     although this is a bad practice.

 The same stuff can be used for packages with
 `\PassOptionsToPackage{<options>}{<package>}`.

### A Word About Fonts

The company style guide often refers to custom fonts. Historically
LaTeX handles fonts via METAFONT system. Fonts from METAFONT give nice
results but most of modern fonts are in OpenType, TrueType or
Adobe Type1 formats. The classic PDF LaTeX unable to handle these
types.

To use such fonts switch to XeLaTeX required. XeLaTeX has native
support of OpenType and TrueType fonts. Another nice feature is
Unicode handling without magic transcoding.

The XeLaTeX can use fonts installed on system by their names or
directly from files.

The first approach is good enough for one time documents. If document
is maintained by many people on different systems the maintaining up
to date font set becomes hard.

The better way is placing fonts under version control along with
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

The simple way to place all classes, packages and other assets like
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

#### Integration With TeXLive/MacTex

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

#### Integration With MikTeX

The `miktex-console` tool handles configuration information. The path
to repository should be added to "TEXMF root directories" list on
"Directories" page in section "Settings".

## Conclusion

The great LaTeX system allows to make perfect documents and
presentations. But creation of maintainable set of documents requires
an extra attention and some diligence. Personally I've found that
recommendations above dramatically simplifies this work.

## Changelog

6 September 2026
: Add sections about handling document class parameters.

27 August 2026
: Add helper command to provide generic access to chapter
  metainformation.

18 August 2026
: * Add information about Beamer customizations.
  * Add hints about bibliographies in Beamer.

9 August 2026
: Added overall process figure and section about TikZ-to-SVG
  conversion.

8 August 2026
: Add:
    * Information about node size issue in TikZ-UML.
    * A note about slide font colors with XeTeX.

5 August 2026
: Fix typos

4 October 2024
: Added:
    * A note.
    * List of "listings" package issues with ranges and line
      numbering.
    * Way of handling `\only` and `\onslide` effects.

24 September 2024
: Add link to Dual-Screen PDF Viewer

29 August 2024
: Initial version.
