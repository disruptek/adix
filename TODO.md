1) While things work well in the tests/\*shell.nim's, should add many tests both
   to exercise everything and ensure a compiler changes don't cause regressions.
   See if we can just use all the one's already in the Nim stdlib.  Add in the
   weak vitanim benchmarks.  Would be nice to do a whole more real suite like
   the probablydance guy & beyond; Unsure I'll have the time/interest.

2) Make a `diset` using sequint, e.g. dcset; Should be easy.

3) Make o?set using sequint, maybe called c?set if separate types or maybe just
   a run-time or compile-time option on o?set.  Should be easy.

4) Add `reindex` proc for o\*set/o\*tab.  Should be easy..just clear idx and
   loop over data[] inserting.  Maybe add after `sort` (to be more drop-in for
   the stdlib's ordered tables).

5) Make `ilset` work with non-zero sentinels.  Very easy;  Kind of the whole point

6) Look into other well vetted integer hashes for althash.

7) Possible lpset/ilset/olset micro-optimizations most relevant for L1/L2 cases:

  a) `moveMem` type element shifting should be faster than current pushUp/etc.,
     especially for larger element sizes.

  b) One thing that might boost some workloads is the idea here:
       https://web.archive.org/web/20170623234417/https://pubby8.wordpress.com/
     The idea is replacing the lower bits of the hash code which recapitulate
     the table index after its computed with the probe depth (after Amble&Knuth
     1973).  The benefit is that the `isUsed and d > depth` check can be folded
     into just one test.

     The main trade off unmentioned in that blog post is that pushUp/pullDn must
     update depths in the back half of the cluster, however large that is and
     one cannot just do (only) memmove shifting.  So, you save 1/3 not so
     predictable branches in the find loop per cluster element, but also create
     like 5 ops per cluster element for mutation operations (load, mask out,
     inc/dec, mask in, store) for the cluster upper half.  The extra ops are
     more predictable work, but very non-negligible compared to the (hc-i)and
     mask kind of ILP work it is saving.  For mutations it is probably close to
     a net wash.  For "read-only after build mostly miss" workloads it could
     help 1.5x (e.g. near empty set intersects).  That's also when Robin Hood is
     *already* a big winner from its half-depth search.  The extra boost could
     constitute good advertising.  Given that memory layout is identical to not
     saving depths, this could also be a run-time/per-tab option { like Robin
     re-org itself is }, not necessarily on by default.  It would even be
     possible to do quick "bulk conversions" of hcodes to combos & back.

  c) Can simplify rawPut/rawDel a lot if assume a strong enough hash by keeping
     an overflow area at the high end of s.data, i.e., s.data is longer than `1
     shl s.pow2` by denom/numer\*s.pow2 ish.  The amount longer will always be
     bounded by that from table resize policy *except* if a hash is so bad that
     this limit is violated with a sparsely full table, which can happen with
     enough probability to be a serious concern.  Graceful degradation is better
     and all we currently do in this overlong at < 50% full circumstance is warn
     or acivate mitigations.  In short, I doubt
     probablydance.com/2017/02/26/i-wrote-the-fastest-hashtable/ makes the right
     safety judgement call for a general purpose table..elsewise a worthwhile
     blog post with less lame benchmarks than usual.  I independently had the
     idea to trigger growth based on probe depth with a memory attack safeguard,
     though.  As did the Rust guys, apparently and surely many in the 1970s.
     It's really a pretty obvious tilt-your-head-the-other-way look at a usual
     probes vs.load graph.  The probablydance guy seems to have missed the
     safeness side of it, though and focused on low loads with good hashes. He
     figured this out in the end, but going w/a short-hop linked variant:
       https://probablydance.com/2018/05/28/a-new-fast-hash-table-in-response-to-googles-new-fast-hash-table/

8) Do external chaining impl for mutating while iterating.  Look at internal
   chain of probablydance to assess insert/delete-while-iterating abilities.

9) Think about adding Cuckoo hashing.  The strong-hash/double fetch trade off is
   unlikely to ever be competitive (in SW) with RHoodLP on large tables with the
   most slowness, but it may appease some people.  DGA seems pretty convinced
   concurrent tables are best satisfied by Cuckoo.  So, it may also be a good
   basis to replace stdlib sharedtables.  There are lock-free LP tables as well.
   So, trust, but verify on this one.

A) It'd be nice if we provided the option to use writable mmaps for all these
   tables instead of just `seq` backing store.  This could maybe be as simple as
   passing an allocator proc to the various constructor functions and doing our
   own pointer arithmetic.  Not too hard, really.

B) Port my C B-tree to Nim and include that here.