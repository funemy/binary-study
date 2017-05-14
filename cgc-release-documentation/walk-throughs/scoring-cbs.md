# Scoring a Challenge Binary Set

This walk-through will guide the reader through the approximate performance
scoring measurements of a challenge binary. This scoring is intended to allow
teams to compare and contrast their approaches to replacement CBs during
development.
**The reader is explicitly cautioned that the DARPA Cyber Grand Challenge
metrics used during events are of _higher precision_ than those contained herein.**

The basic formula for scoring is:
    score = Availability * Security * Evaluation

The security and Evaluation terms are described in the Cyber Grand
Challenge FAQ[[2]][ref2] and CQE Scoring Document[[1]][ref1]; they
will not be reproduced here.  The availability score is a combination
of the retained functionality (number of successful polls) and the
performance.

## Terminology

Cyber Reasoning System (CRS): the automated systems competing in the Cyber
Grand Challenge.

Challenge Binary (CB): an individual binary which is a part of a Challenge
Binary Set.

Challenge Binary Set (CB Set): the set of binaries which comprise a single
challenge (will not be an empty set)

Unpatched Reference CB Set: the original set of binaries comprising a single
challenge set that are provided to all CRS.

Patched Reference CB Set: a CB set containing patched for the
vulnerability(ies) in the Unpatched Reference CB Set.

Replacement CB Set: a set of binaries provided by a CRS to the CGC
organizers.

All metrics are computed as a comparison between a Replacement CB Set
and the Reference CB Set.  The CGC team will compare CRS using the
Patched Reference CB Set, but competitors can approximate the scoring
metrics using the Unpatched Reference CB Set.

## Performance

There are many methods by which the performance of a CB Set could be
measured. For the sake of this walkthrough, only 3 readily available
metrics will be considered:

1. ExecTimeOverhead: user time (task clock)
2. MemUseOverhead: maximum resident set size (maxrss) and number of minor faults (minflt)
3. FileSizeOverhead: file size

Task clock is available through the Linux-specific performance monitoring
API (perfmon).  Maxrss, and minflt are available via wait2()'s rusage pointer
for an individual CB.  The cb-server program computes the task clock, maxrss,
and minflt for a CB set for each poll by summing the corresponding
measurement for each individual CB in the set for the given poll.

A CRS may sum statistics from the polls for both a reference and
replacement binary set to compute a relative performance measure for
each statistic.

For example, suppose a Reference CB Set contains 4 CBs and 100 polls are
available for testing.  cb-test [[3]][ref3] can be called for each CB Set
and will produce 100 separate task clock values (one for each poll).
Let the REFtaskclock = sum(the 100 task clock values from the reference
CB set).

A replacement CB set built by the CRS might be measured the same way
using cb-test, which will produce an additional 100 task clock values.
Let REPtaskclock = sum(the 100 task clock values from the replacement
CB set).

The ExecTimeOverhead would be computed as follows:
    ExecTimeOverhead = REPtaskclock / REFtaskclock - 1.

The same algorithm could be used to yield a relative memory usage metric: 
    MemUseOverhead = 0.5 * (REPmaxrss / REFmaxrss + REPminflt / REFminflt) - 1.

Similarly, the file size of a CB Set is the sum of all CBs in the set:
    FileSizeOverhead = size(replacement CB set) / size(reference CB set) - 1.

## EXAMPLE

    cd /usr/share/cgc-sample-challenges/examples/LUNGE_00002
    sudo make generate-polls
    cb-test --debug --cb LUNGE_00002 --xml_dir poller/for-release/ \
      --directory bin --log /tmp/LUNGE_00002.for-release.txt

When the above completes (after approx 1.5min), the log file (specified with --log) will
contain entries from cb-server [[4]][ref4] of the form:


    # cb-server: total children: 1
    # cb-server: total maxrss 3336
    # cb-server: total minflt 834
    # cb-server: total utime 0.010000
    # cb-server: total sw-cpu-clock 1820594
    # cb-server: total sw-task-clock 1820801
    ...
    # cb-server: total children: 1
    # cb-server: total maxrss 3344
    # cb-server: total minflt 901
    # cb-server: total utime 0.020000
    # cb-server: total sw-cpu-clock 2543666
    # cb-server: total sw-task-clock 2592890

One set of records will be created for each test case showing:
* the number of binaries in the Challenge Set (1)
* the sum of all binaries in the CB Set's maxrss (3336)
* the sum of all binaries in the CB Set's minflt (834)
* the sum of all binaries in the CB Set's task clock values (1820801)

These can be aggregated and results used for low-fidelity performance
comparisons.


## SEE ALSO

For information regarding the building a CB, see the building-a-cb.md
walk-through [[5]][ref5]

[1] [Cyber Grant Challenge: Frequently Asked Questions(FAQ)][ref1]

[ref1]: https://cgc.darpa.mil/documents.aspx "Cyber Grand Challenge: Frequently Asked Questions (FAQ), July 24, 2014."

[2] [DARPA Cyber Grand Challenge: CQE Scoring Document][ref2]

[ref2]: https://github.com/CyberGrandChallenge/cgc-release-documentation/blob/master/CQE%20Scoring.pdf?raw=true "DARPA Cyber Grand Challenge: CQE Scoring Document"

[3] [cb-test(1): Challenge Binary testing utility][ref3]

[ref3]: https://github.com/CyberGrandChallenge/cb-testing/blob/master/cb-test.md "cb-test(1): Challenge Binary testing utility"

[4] [cb-server(1): Challenge Binary launch daemon][ref4]

[ref4]: https://github.com/CyberGrandChallenge/service-launcher/blob/master/cb-server.md "cb-server(1): Challenge Binary launch daemon"

[5] [Building a Challenge Binary][ref5]

[ref5]: https://github.com/CyberGrandChallenge/cgc-release-documentation/blob/master/walk-throughs/building-a-cb.md "Building a Challenge Binary"

For support please contact CyberGrandChallenge@darpa.mil
