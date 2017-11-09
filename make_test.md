# Making `make test` pass

I've spent some time getting Riak's `make test` command to
run reliably. The TL;DR is that one can now run `make test` on a top-level Riak clone
and all the tests pass.

## Why?

Another question might be: why did a clean clone and run of `make
test` on the released Riak-2.2.3 tag fail? I don't know the answer. I
don't think it matters anymore, either. Given the disorderly
dissolution of Basho the projects are in surprisingly good shape.

If Riak is going to continue to exist as an open source, community run
project, it must be possible to contribute, and to do that there must be
a trustworthy test suite. Without addressing the merits of the current
test suite, it should at least pass deterministically on the currently
shipped code, and then we can improve on it.

## Why now?

Since Basho's demise, [bet365](https://twitter.com/bet365Tech), the
[NHS](https://twitter.com/nhsdigital),
[Erlang Solutions](https://www.erlang-solutions.com/products/riak.html),
[Tiot.jp](https://www.tiot.jp/en/solutions/riak/), and other
organizations have been working towards a new release of Riak. At the
[Riak meet up (StokeCon2017)](https://www.meetup.com/RIAK-Development-Roadmap-Workshop/events/243302656/)
it was [agreed](http://bet365techblog.com/riak-workshop-summary) that
the first community release of Riak should be a low risk affair. The
aim is to show that we can release Riak, and run what we release in
production. We decided to go back to the "last known good" release,
[Riak-2.2.3](http://docs.basho.com/riak/kv/2.2.3/release-notes/),
rather than review all the changes committed since then. Our aim is to
release a Riak-2.2.5 by the end of 2017. This release will be based on
Riak-2.2.3, and will have minor additive feature changes, and some
small bug fixes. It is intended to have value, but a lot of the value
comes from releasing our first version of Riak as a community. We plan
on bigger things for Riak-3.0 in 2018, but walking comes before
running.

In order to get the release wheels rolling we need a reliable test
suite.

## What did you do?

I cloned Riak, checked out the Riak-2.2.3 tag, and ran `make
test`. Did it pass? No, it did not. That first run 16 of Riak's ~50
dependencies failed. Not terrible, but not ideal either.

Two dependencies, Riak Search and Merge Index (a dependency of Riak
Search), have been deprecated since Riak-2.0. Rather than fix the
failing tests, I excised old Riak Search.

We began this work prior to the bet365 Basho IP purchase announcement,
so we forked Riak into <https://github.com/nhs-riak/riak> and set to work
understanding, and fixing each failure. Riak has a sizable collection
of Quickcheck tests, which have been extremely valuable, but do
require a license from Quviq to develop and run. Quickcheck is
property based testing with random generation. Some of the failures
only occurred due to luck/probability. There may well be others
lurking.

There were 3 main types of failure:

1. Path errors - Riak's deps are projects in their own rights, and
   many expect to have their tests run a certain way.

2. Trivial fixes - some libraries had small errors in the test code or
   code under test, some of these are even fixed in later versions,
   and were small changes, easy to verify.

3. Non-deterministic failures - some libraries have/had genuine bugs
   in either the tests or code, these were the hardest to address.

For 1 and 2 the fixes were often self evident. For each such library
we forked it, created a new branch based of the tag released with
riak-2.2.3, named for the tag and "nhs-riak.2.2.5", and fixed the test
issue. For example, Yokozuna, Riak's Solr integration, has a
Quickcheck test whose post-conditions were incomplete. The tag for
Yokozuna for Riak-2.2.3 was 2.1.10. We created a new branch
`rdb/2.1.10-nhs-riak-2.2.5` and fixed the Quickcheck test. Any
`rebar.config` files that reference the dependency must also be
updated. In this way quite a few dependency projects ended up being
forked and new branches created, sometimes just to update a
`rebar.config` file.

Some of the failures in category 3 could be fixed, others were more
stubborn. [Sidejob](https://github.com/basho/sidejob) Basho's bounded
mailbox library was the most problematic. Before going into detail
about what is wrong with sidejob, we elected to "fix" it be disabling
one test that could _never_ pass with the current design of sidejob,
and fuzzing some others to take into account the real properties of
sidejob. This is not ideal, and of course no one is happy about
this. We raised an [issue](https://github.com/basho/sidejob/issues/17)
against the repository for future tech debt work. If sidejob has been
shipped in this state for at least 3 releases of Riak, and as yet no
bug has been raised, one could argue that we're not making anything
worse by slackening the test constraints, whatever, it feels bad<sup id="a1">[1](#f1)</sup>.

## What's not done?

- Sidejob isn't "fixed".

- Riak hasn't been updated to run on a modern Erlang.

- Riak hasn't been updated to use rebar3.

The reason being, as stated above, that we want Riak-2.2.5 to be as
low-risk as possible, whilst still having an incentive to deploy the
new version. Success is built on success, and momentum builds, waiting
for the ideal could well be enough to kill the fledgling community
project. We are committed to tackling these major concerns for the
following Riak-3.0 release. If you want to be involved, please join
the
[mailing list](http://lists.basho.com/mailman/listinfo/riak-users_lists.basho.com),
[#riak on Freenode IRC](http://irc.lc/freenode/riak), and/or
[Slack](https://postriak.slack.com/).

## What's next?

Riak also has a sizable integration testing suite called
[riak-test](https://github.com/basho/riak_test). Now we're in the
process of setting up repeatable runs of this suite against the new
Riak-2.2.5 resulting from the `make test` work. When they pass
dependably we'll add in the Riak-2.2.5 feature changes and
fixed. After that, it's build and release time.

This has not been the most interesting or glamorous work. I've been
paid for doing it. I am in awe of those in the open source community
who undertake this kind of task on their own time.

It would be great if as many people as possible were to run `make
test` and let me know about any issues.

    git clone https://github.com/nhs-riak/riak.git
    cd riak
    git checkout rdb/nhs-riak-2.2.5
    make locked-all
    make test

You can get in touch with me via slack, IRC, or the mailing list and
let me know how it went. Many thanks!

------
UPDATE: I just added the newly open sourced (thanks bet365) riak_repl, riak_snmp, and riak_jmx (AKA Enterprise Riak) deps to the NHS fork, and the tests pass.
------


<b id="f1">1</b> sidejob may well not be broken at all. The library
author may have intended sidejob to be this way, and the test author
may not have known. There are no comments, or documentation, so we're
left to think the best of all parties involved and assume sidejob
works as designed, with a trade-off of correctness for performance,
and the test authors did not know of this. It is part of the lost
history of Basho.[↩](#a1)
