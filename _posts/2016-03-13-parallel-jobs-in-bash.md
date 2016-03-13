---
layout: post
title: "Parallel jobs in Bash"
date: 2016-03-13
categories: coding
tags: bash
---
{::comment}
The filename must be on this format:
2010-01-01-some-sort-of-title.md
the title in the header will be the one displayed on the web pag.
{:/comment}
Sometimes you need to run a large number of jobs in a shell.
Running these jobs in parallel can save a lot of time, 
but can be a bit tricky to set up.er of jobs in a shell. Running these jobs in parallel can 
This post will cover a few tricks I often use to get quickly get things done.

## A simple example running in serial
Consider the following shell script:

~~~ bash
#!/bin/bash
ARG_LIST="10 10 10"
for i in $ARG_LIST; do
    sleep $i    # Main job
done
echo "Time elapsed: ${SECONDS}s"
~~~
In this example, we invoke **sleep** three times,
which basically waits for a given amount of time.
How long we wait is determined by the argument,
which in our case is 10 (seconds) at each invocation.
At the end of the script we print the number of seconds elapsed 
since the script started running,
this is typically around 10x3=30s.

## Background jobs
The first method for parallelization which many shell programmers encounter
is to simply run jobs in the background.
This is easiest invoked by putting a '&' after the command.

~~~ bash
#!/bin/bash
ARG_LIST="10 10 10"
for i in $ARG_LIST; do
    sleep $i &  # Main job
done
wait    # Wait for all background jobs to finish
echo "Time elapsed: ${SECONDS}s"
~~~
Running this script will invoke three background instances of the **sleep** command.
Once the background jobs are started, we invoke **wait**,
which will wait for all background jobs to finish.
Each instance of **sleep** will wait for 10 seconds,
but since we run the jobs in parallel,
the whole script will only takes about 10s to finish.

This method of running jobs is useful if you need to run a small number of jobs
and you have a couple of idle cores which can be used.

## A larger number of jobs
What if the number of jobs exceed the number of free cores on your system?
If all we run is **sleep**, then we're probably fine, 
since the CPU doesn't have have to work while **sleep** is "running".
However, if we replace **sleep** with a command that actually _does something_,
then we will quickly run into problems
when the number of jobs exceeds the number of CPUs.
In order to counter this, we can modify our previous example,
by initiating jobs in smaller batches:

~~~ bash
#!/bin/bash
N_CPU=4             # Number of CPUs available
N_RUNNING_JOBS=0    # Number of background jobs
ARG_LIST="10 10 10 10 10 10 10 10"
for i in $ARG_LIST; do
    sleep $i &	# Main job
    let N_RUNNING_JOBS++    # Increase counter
    if [[ $N_RUNNING_JOBS -ge $N_CPU ]]; then
        wait    # Wait for current batch to finish
        N_RUNNING_JOBS=0    # Reset counter
    fi
done
wait	# Wait for remaining jobs to finish
echo "Time elapsed: ${SECONDS}s"
~~~
In this example we run a total of 8 jobs,
but the jobs are launched in batches of four.
When four jobs have been launched,
the script will wait for these jobs to finish before launching the next batch.
This example would take about 10x2=20 s to finish.
Running the same jobs in serial would take 10x8=80 s.
When we get up to very large numbers of jobs,
these tricks really begin to matter!

## Variable processing times
In the previous examples,
we ran a number of jobs which all took the same amount of time to finish;
that is not always the case.
Let us go back to our first example,
increase the number of jobs from 3 to 8,
and modify the time each job takes:

~~~ bash
#!/bin/bash
ARG_LIST="15 5 19 1 13 7 14 6"
for i in $ARG_LIST; do
    sleep $i    # Main job
done
echo "Time elapsed: ${SECONDS}s"
~~~
These jobs will take a total of 80 s to finish when run in serial.
If we use this **$ARG_LIST**,
in the example from the previous section,
we would have a total processing time of 33 s
which is quite far from the 20 s of the previous example.
This is because the time is determined by the longest running job in each batch,
i.e. 19+14=33 s.
This means that most cores spend a large portion of their time doing nothing
(well, almost _all_ of the time,
since our main job is **sleep**,
but let's not think about that for now).

One method to reduce the idle time is to further adjust our script
to expect jobs to finish at different times.
We can do this by replacing the if-statement
with a while-loop:

~~~ bash
#!/bin/bash
N_CPU=4             # Number of CPUs available
N_RUNNING_JOBS=0    # Number of background jobs
ARG_LIST="15 5 19 1 13 7 14 6"
for i in $ARG_LIST; do
    sleep $i &	# Main job
    let N_RUNNING_JOBS++    # Increase counter
    while [[ $N_RUNNING_JOBS -ge $N_CPU ]]; do
        sleep 0.5   # Wait for .5s
        N_RUNNING_JOBS=$(jobs -r|wc -l) # Get number of running jobs
    done
done
wait	# Wait for remaining jobs to finish
echo "Time elapsed: ${SECONDS}s"
~~~
The main difference here is that
our script will regularly check the number of jobs
running in the background. When there are four (or more) jobs running, we wait;
when there are 0-3 jobs running, we launch new jobs.
This example takes about 26 s to finish.

This is what happens:
: At time (t) = 0 The first four jobs (15, 5, 19, 1) are started.
: At t=1 s, the first jobs has finished
and the script launches the fifth job (13).
: At t=5 s, the second job has finished
and the script launches the sixth job (7).
: At t=12 s, the sixth job has finished
and the script launches the seventh job (14).
: At t=14 s, the fifth job has finished
and the eight job (6) is started.
: The main loop exits
and the script will now wait for the remaining four jobs (15, 19, 6, 14) to finish
: At t=15 s, the first job has finished.
: At t=19 s, the third job has finished.
: At t=20 s, the eight job has finished.
: At t=26 s, the seventh job has finished.

So it's still not perfect, but it's better.
The "extra" 6 s of run time comes from the time at the end,
when the job queue is empty.
Re-ordering the job list could in our example reduce the processing time to 20 s.
However, we might not know how long time each job will take
when setting up our script.
Regardless,
compared to the example in the previous section,
this last example will always be equally or more efficient,
especially when running a large number of jobs.

## Final remarks
These examples are pretty much part of my everyday workflow by now,
but there are plenty of other approaches.
I recently learned of 
[GNU parallel](https://www.gnu.org/software/parallel/);
which is probably a much cleaner method if you need a bit more functionality.

For speeding up data analysis I prefer to use the python package
[multiprocessing](https://docs.python.org/2/library/multiprocessing.html),
maybe I'll write a post about that in the future.

If you need really advanced features, with good performance,
it might be worth looking into 
([MPI](http://www.mpi-forum.org/) or
[OpenMP](http://openmp.org/));
but these are much more complicated than the simple examples I've shown here.

