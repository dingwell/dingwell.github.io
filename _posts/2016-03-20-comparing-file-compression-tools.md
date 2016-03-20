---
layout: post
title: "Comparing file compression tools"
date: 2016-03-20
categories: coding
tags: bash
---
{::comment}
The filename must be on this format:
2010-01-01-some-sort-of-title.md
the title in the header will be the one displayed on the web page.
{:/comment}


In many of my projects I end up with several TBs of data.
I rarely get back to it once the project is finished
but the data needs to be archived nonetheless,
just in case it is later requested.
It is quite uncommon that I reuse data from finished projects
and when I do it is usually only small subsets;
therefore, it often makes sense to compress most of it.
But compressing Terabytes worth of data takes time
and sometimes the compression ratios are not worth the effort.
With the numerous compression tools available,
it is not surprising to find plenty of reviews/comparisons on the web.

The problem is that when working with less common data formats,
or scientific data,
compression performs very differently compared to "normal" use cases.
Because of this, I often prefer to run the tests myself,
but that can take a lot of time and effort.
So, I wrote a script which compares a number of different compression tools.
I typically run this on a subset of my data
before choosing which tool to use for the lot.
I recently got back to it and cleaned it up bit
and finally [posted it on github](https://github.com/dingwell/everyday_benchmarks/blob/master/test_compression.sh).
It formats the output to look nice in a terminal
and work well with some flavours markdown.
It assumes that you have gzip, bzip2, lzop and xz installed on your system.

Here is an example using some GRIB1-data
(sorry about the slightly confusing format,
my current theme does not handle tables very well):

|--------+----------------+--------+--------+--------+-------|
|        |                |  gzip  |  lzop  |  bzip2 |   xz  |
|:------:|:---------------|-------:|-------:|-------:|------:|
|LEV1    | COMP. RATIO    |  30.8% |  22.9% |  29.4% |  32.4%
|        | COMP. TIME [s] |      3 |      1 |     17 |     19
|        | DEC.  TIME [s] |      2 |      3 |      7 |      7
|------------------------------------------------------------|
|LEV6    | COMP. RATIO    |  30.9% |  23.0% |  34.3% |  33.1%
|        | COMP. TIME [s] |      4 |      1 |     16 |     27
|        | DEC.  TIME [s] |      2 |      3 |      7 |      7
|------------------------------------------------------------|
|LEV9    | COMP. RATIO    |  30.9% |  29.3% |  34.8% |  33.1%
|        | COMP. TIME [s] |      5 |     11 |     17 |     26
|        | DEC.  TIME [s] |      1 |      0 |      7 |      6
|------------------------------------------------------------|

'RATIO' above is the amount of space _saved_ relative to uncompressed files.
The total file size before compression was 145 MB.

I will not discuss the results of this example, instead,
I encourage you to try for yourself with your own set of files;
that's the only way to ensure that you pick the best solution for you.
