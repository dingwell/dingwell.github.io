---
layout: post
title: "WRF kF_ETA errors and possible solutions"
date: 2017-11-30
categories: modelling
tags: wrf
---
{::comment}
The filename must be on this format:
2010-01-01-some-sort-of-title.md
the title in the header will be the one displayed on the web pag.
{:/comment}

This is a short reference of possible causes for the kf-eta-related errors:
~~~
WOULD GO OFF TOP: KF_ETA_PARA ...
~~~
There are many suggested solutions for these but I always manage to find some new way to
produce these crashes.


## Commonly suggested causes/solutions
I've found these solutions listed on various web-pages
* Move domain boundaries (try to avoid steep topography)
* Decrease time-step (if this is an isolated event in a longer run, try to
  increase the time-step again after the next restart interval)
* Turn on **w\_damping** (tries to reduce vertical winds)
* Change PBL-scheme (I've never tried this)

## Less commonly suggested causes/solutions
* Change microphysics (I've never tried this)
* Turn _off_ **w\_damping** (?!) might work in some cases
* Disable optimizations at compile-time (set FCOPTIM=O1 or = O0 in
  configure.wrf)
* Buggy metgrid tables (check met\_em\*-files for possible errors)

## Causes/solutions that worked for me
* **Bad input data: pressure.**
  E.g. when forcing data is on sigma levels but METGRID.EXE didn't read the
  PRES\*-files(!). Then met\_em\*-files will not have the correct pressure data.
  At the lowest model layer everything looks normal but all layers above that
  simply have the inverse level number as pressure;
  this causes all kinds of weird behaviour in WRF, especially for the vertical
  wind.
* **Two-way nesting**
  Can cause weird feedback patterns near domain boundaries.
  I've usually found the cause of the problem to be something else but if there
  are no other obvious causes,
  it could be worth testing to spin-up the inner domains before turning on the 2-way
  nesting...

## Finding the root of the problem
* Look for unrealistic values the input data in e.g. ncview.
* Decrease the output time-step to about a minute and see if anything
  strange happens with before the crash.
* Does the error always occur in the same point? If so, look at input/output
  values in that point


## Further reading
* [WRF-user forums](http://forum.wrfforum.com/viewtopic.php?f=6&t=263)
