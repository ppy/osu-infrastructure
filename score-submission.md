# Score submission infrastructure

## Introduction

This document aims to cover the current structure of score submission from an infrastructure perspective, with the goal of moving towards consolidating the future (lazer) and present (osu-stable) into some kind of combined leaderboard.

## Goals / Timeline

The plan going forward is to leave as many options available to us as possible, but also to fix existing storage backends that no longer offer the flexibility we require due to the sheer amount of data being stored. The main example of this is MySQL, where table changes can take weeks to run, and come with an overhead which can cripple infra-wide performance.

A vague plan of how I plan on moving forward with getting solo leaderboards working in lazer is:

### Add basic solo submission flow to lazer client

This was mostly already implemented thanks to the work done on the playlists and multiplayer systems. It has already been implemented and deployed in the last release, but not yet tested for correctness.

### Add basic score storing to osu-web

Again, this has been build on top of the existing logic for playlists / multiplayer. This is deployed and there are tables ready to store scores from lazer (although on a quick check they don't seem to be correctly arriving yet).

### Confirm data is flowing from client to server and storing correctly <- CURRENTLY HERE

Need to test lazer locally to confirm that the web request logic is correct, as the server-side components weren't implemented when I wrote / merged the code to handle score submission.

### Begin to split off data processing where feasible (focus on basic statistics updates)

Basic statistics which can be updated:

- Play count
- Level
- Total score
- Hit counts
- Recent plays

Once the above has been completed, we are ready to start consuming the most basic data from this flow. The best example here would be updating the playcount and hit count statistics based on incoming scores from lazer. This could either be implemeneted in `osu-web`, or via a queue processor (using `osu.Server.QueueProcessor`).

Further consideration needs to be given as to which statistics we can increase and how much consistency check logic needs to be present to make this happen. There are a lot of checks in the legacy score submission code to ensure users are not abusing the systems, so a minimal number of these will need to be reimplmemented based on which statistics we decide to update and how important consistency on those stats are.

### Figure out storage method for forward storage

We probably don't want to be storing scores in the way we are right now. Whether we switch to a NoSQL backing or otherwise, some futher investigation needs to be done as to how we can ensure we don't bottleneck ourselves going forwards.

I already have some thoughts as to how we can make some huge improvements, based on the current pain points which are holding us back in scaling and implementing new features.

### Implement new storage and required server components

This will involve deploying new infrastructure at a scale which can handle the full existing score submission load. It may also involve streaming the full load of incoming (stable) scores into the new storage to stress test it.

### Start to save scores from lazer to new storage

Implement the rest of the pieces required to get scores going into the new storage. Again, this may happen in osu-web itself or as an external component.

At this point we probably want to have some kind of anti-cheat ready in lazer, as well.

### Import stable scores to new storage

Start a bulk import of all existing scores across to the new storage. This should be done in a way that it can be re-run at any point, and re-runs should ensure consistency with existing data. I guess in a similar way to ElasticSearchIndexer project.

### Implement and test score wiping (in case we need to remove scores for all lazer or specific lazer, etc.)

We should be able to, given one or more client version hashes, remove all related scores (and rollback relevant stats). This is something we haven't been able to do until now, which has limited the ability to have test runs of game-breaking changes or to limit rollback of bugs to only those scores submitted against the affected version(s).

Make sure that this process can be run efficiently regardless of the size of the wipe.

### Balancing, forward plans, etc.

There's a lot to discuss in terms of balancing and integration of the two score sources. Some that come to mind are:

- we need to figure out how the two methods of scoring are going to work together going forward
- smoogipoo wants to discuss the concept of "prestige" leaderboards
- i want to abolish mod multipliers

## Current infrastructure

### Process

The majority of the score submission process is currently synchronous. Going forward the plan is to split things out into individual queue processors for different pieces of the puzzle, and keep the ingest as simple as possible to ensure high throughput and ease of scalability.

Firstly, let's look at all the things which currently happen in the average score submission:

- osu-stable sends a request to osu-web-10
- osu-web-10 receives the score
  - checks user authentication

  - queues score for validity check (osu token processor queue)

  - foreach spotlight / ranking target
    - update basic stats and store 24h rolling score entry
    - if the score is a pass
        - check medal unlocks
        - if the score is a new user high
          - store permanently in high scores table
          - wait for x ms for pp to be populated by `osu-performance`
            - `osu-performance` is constantly polling for new scores and usually processes the pp before timeout
  - return updated statistics to the user, one row per leaderboard, including pp if available

### Statistics

For simplicity, these stats are *just for the osu! ruleset*. The overall number is around 30-40% higher than this.

24h Scores (stored regardless of pass/fail)
---------

Scores ingested: 15,579,568
Ingested scores which are non-fail: 3,268,037
Storage size: 3.7gb
Size per score: 237 bytes

Overall High Scores
---------

Scores stored: 1,250,672,940
Storage size: 195gb
Size per score: 167 bytes
