# Score infrastructure migration plan

This document aims to cover the current structure of score submission from an infrastructure perspective, with the goal of moving towards consolidating the future (lazer) and present (osu-stable) into some kind of combined leaderboard.

This is a second version of the document, written from the perspective of being at a point of already having defined the main storage structure for scores going forward (the main storage table `solo_scores`, but with many of the surrounding infrastructure changes not yet being completed (elasticsearch schema changes, osu-web support, medal processor updates etc.).

## Shortcomings of current system

### Score ID spaces overlap for different rulesets

Because we store each ruleset's scores in a separate table with `autoincrement`, scores from different ruleset may have the same ID. Additionally, as we eventually plan to introduce new rulesets this limits the scalability of the system.

### Lookups are slow for non-indexed data

The existing high score tables are limited in efficiency by the indices that exist on them. These have been decided with beatmap leaderboards in mind, and are relatively efficient for that specific purpose.

```sql
KEY `user_beatmap_rank` (`user_id`,`beatmap_id`,`rank`), 
KEY `beatmap_score_lookup_v2` (`beatmap_id`,`hidden`,`score` DESC,`score_id`)
```

This is alleviated for user profiles by using the elasticsearch server to do `(user, pp desc)` lookups.

The major issue is country and mod filters, which have no index and result in expensive table scans. Efficiency drops as the filters become more specific, as both can still use the `beatmap_score_lookup_v2` index as a precondition before performing the scan operation (and queries generally request the top x results, so lookups which find rows early in the scan are faster).

Going forward, we are likely dropping mod multipliers and removing the supporter requirement for mod specific filtering (at least to some extent) to allow filtering classic scores only, for instance. The performance of such queries needs to be magnitudes better than what it is right now.

### Adding new attributes to data is slow

If we need to make changes to score storage, online `ALTER TABLE` operations are required. This can take days to run (`osu_scores_high` is sitting at 230gb / 1.517 billion rows), during which the overhead of having triggers for online replication can cause enough extra load to cause infrastructure-wide delays.

### Updating PP during recalculations causes high read-write contention

Because pp values are stored in the main score table, updating these can be considered one of the most expensive things we can do. With each index on a table updates get slower, but the additional kick-in-the-dick is that this is the most accessed table in the system, so these writes can cause additional delays.

While there is enough head-room on the database servers to handle this load from an IO/CPU perspective, the killer comes from replication. Both the addition data flow and relatively single-threaded nature enforced by replication means that slave databases can fall behind if PP values are written too fast. Due to this, we artificially limit the speed of updates to ensure that slaves don't fall behind (which can in turn cause leaderboards or score submission to not have the timely data they expect).

### Score submission is synchronous

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

## Goals / Timeline

The main goal is to complete this change before the primary key address space runs out for `osu_scores_high` (we currently use `UNSIGNED INT`). The overhead of converting this to a larger `BIGINT` type will be a pain to deal with, so moving away to the new structure and avoiding this altogether is preferred.

Currently we are sitting at 4,048,822,061 at a usage rate of 1.3 million per day. This gives us 189 days as of 2022-02-01, so all usage of the old tables should stop well before 2022-08-01.

## New infrastructure

This section outlines each piece of the new infrastructure which needs to come online, in a roughly chronological order to allow for a both systems to operate in parallel for a period of time. This will allow us to ensure nothing has been forgotten, and potentially make changes (or reinitialise the new system from scratch) if required with no impact on the existing infrastructure.

### ✅ Add basic solo submission flow to lazer client

osu!(lazer) is already submitting scores via an osu-web endpoint. These are already being stored to the new `solo_scores` table.

Note that as these stores are not displayed anywhere, they are expendable. We can still reset the `solo_scores` table completely and not worry too much about these lazer-first scores. The only consideration is that if we do this, we cannot rollout new statistics processors on these scores. From that angle, we may want to avoid doing this until components like the medal processor have been rolled out.

### ✅ Add basic score storing to osu-web

As above, osu-web is already capable of storing scores to the new infrastructure.

### 🏃 Implement statistics processor components to update user profiles and history

Basic statistics which can be updated:

- Play count
- Level
- Total score
- Hit counts
- Recent plays

Not yet supported:

- Medals
- Play time

### ✅ Decide on storage method and structure

We have already finalised the structure of the new `solo_scores` table as follows:

```sql
CREATE TABLE `solo_scores`
(
    `id`         bigint unsigned    NOT NULL AUTO_INCREMENT,
    `user_id`    int unsigned       NOT NULL,
    `beatmap_id` mediumint unsigned NOT NULL,
    `ruleset_id` smallint unsigned  NOT NULL,
    `data`       json               NOT NULL,
    `preserve`   tinyint(1)              DEFAULT '0',
    `created_at` timestamp          NULL DEFAULT NULL,
    `updated_at` timestamp          NULL DEFAULT NULL,
    `deleted_at` timestamp          NULL DEFAULT NULL,
    PRIMARY KEY (`id`),
    KEY `solo_scores_user_id_index` (`user_id`),
    KEY `solo_scores_preserve_index` (`preserve`)
);
```

- The majority of scoring data has been moved into the `data` JSON field. The structure of this JSON is documented in [osu](https://github.com/ppy/osu-queue-score-statistics/blob/82ba5f48ca6b268b52b028ba70918db2fee09382/osu.Server.Queues.ScoreStatisticsProcessor/Models/SoloScoreInfo.cs#L21) and [osu-web](https://github.com/ppy/osu-web/blob/master/app/Models/Solo/Score.php). The schema of this JSON is intended to evolve over time, but should also be backwards compatible (ie. we should only be adding data, in general, unless we are also performing migration).
- All columns in this table that are duplicated redundantly outside of `data` are done so for efficient lookup purposes.
- The `preserve` flag marks whether the score in question should remain in the database. When we have purge logic in place, scores which are not marked as `preserve` will be deleted permanently x days after `updated_at`.

Further consideration:

- We likely will need to encompass `beatmap_id` (and maybe `ruleset_id`) into the `user_id` index for internal lookup purposes (ie. when a user sets a score, we will want to compare other scores that same user has set on the beatmap to resolve any conflicts and decide on a deletion strategy). This requires some profiling on whether increasing the length of the key has an adverse effect on performance metrics we care about.
- We likely don't need the full laravel `timestamp` column specifications (specifically `deleted_at` and potentially `created_at`, although this one is arguable as we may require `updated_at` to track the last usage for purging non-`preserve`d scores). Removing these could reduce storage requirements.
- The `preserve` flag will probably need to be managed by `osu-web`. For instance, a score may need to be preserved due to a user having it pinned to their profile, regardless of its usage elsewhere.

```sql
CREATE TABLE `solo_score_tokens`
(
    `id`         bigint unsigned NOT NULL AUTO_INCREMENT,
    `score_id`   bigint               DEFAULT NULL,
    `user_id`    bigint          NOT NULL,
    `beatmap_id` mediumint       NOT NULL,
    `ruleset_id` smallint        NOT NULL,
    `build_id`   mediumint unsigned   DEFAULT NULL,
    `created_at` timestamp       NULL DEFAULT NULL,
    `updated_at` timestamp       NULL DEFAULT NULL,
    PRIMARY KEY (`id`)
);
```

- Tokens are temporary entries which mark the beginning of a new play. The primary purpose is to store information which is provided by the client at the beginning of the play which may not be conveyed again at final submission (or may be preferred to arrive sooner for validation purposes). It can also be used to obtain the wall clock time passed between start and end of the play (including pauses or delays in score submission).

```sql
CREATE TABLE `solo_scores_performance`
(
    `score_id` bigint unsigned NOT NULL,
    `pp`       double(8, 2) DEFAULT NULL,
    PRIMARY KEY (`score_id`)
);
```

- Performance point values have very intentionally been stored separate from the scores themselves. This will allow for updates to be performed without creating any contention on the main storage table, but also flexibility to potentially:
    - Have multiple tables during pp updates, with `osu-web` falling back to old tables if data is not populated in the newer one. This can allow for insert-only pp population.
    - Have calculations performed offline during pp updates, and dropped in-place once completed (note that this would require a catch-up process or merge process, but could potentially be the most efficient way of performing pp updates).
- This data will likely be flattened in actual usages alongside other pieces from `solo_scores` (ie. in elasticsearch indices).

```sql
CREATE TABLE `solo_scores_process_history`
(
    `score_id`          bigint    NOT NULL,
    `processed_version` tinyint   NOT NULL,
    `processed_at`      timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`score_id`)
);
```

- Tracks processing of scores, currently by [osu-queue-score-statistics](https://github.com/ppy/osu-queue-score-statistics) exclusively. Allows for changes in processing to be reapplied to existing scores (as the processor can handle reverting and reapplying based on the `processed_version`).

### ⏱ Create score ID linking table

Scores in the new `solo_scores` tables have new IDs. We need a table structure to link old scores to the new ones, for cases where a request is made against the ID directly (ie. a [score display page](https://osu.ppy.sh/scores/osu/4049360982)).

Current proposal:

```sql
CREATE TABLE `solo_scores_legacy_id_map`
(
    `ruleset_id`   smallint unsigned NOT NULL,
    `old_score_id` int unsigned      NOT NULL,
    `score_id`     bigint            NOT NULL,
    PRIMARY KEY (`ruleset_id`, `old_score_id`)
);
```

### ⏱ Create a new elasticsearch schema

Currently elasticsearch is used for user profile pages to get "user best" scores. Given that all score metadata is already loaded into elasticsearch, we could actually have been using it for more than this (taking some serious load off the database servers).

With the new table structure, the above becomes a *requirement*. All leaderboard lookups will need to be done via elasticsearch as there will be no means (index) to do so via mysql.

### ⏱ Update osu-web to display scores using the new structure

As we are going to be running both systems alongside each other, the ability to display scores from the old and new table structure is required. Current proposal is:

```
https://osu.ppy.sh/scores/osu/4049360982 <- old
https://osu.ppy.sh/scores/4049360982     <- new (doesn't require ruleset prefix)
```

### ⏱ Create a pump and ES population flow

Scores coming in via `osu-web-10` will need to populate into `solo_scores` in real-time. When this happens, we will also need to ensure that ES is made aware of new scores. Historically this has been done using the
[osu-elastic-indexer](https://github.com/ppy/osu-elastic-indexer) component – whether we update this to work with the new table or replace it with, for instance, hooks installed in osu-web API endpoints is yet to be decided.

### ⏱ Decide on purge mechanism for `solo_scores`

As mentioned previously, we will want to clean up the `solo_scores` tables in the case the `preserve` flag lets us know that a score is not being used anywhere.

Traditionally, non-high scores (ie. when a user's score did not replace their existing score on the same beatmap-ruleset-mod combination) would be inserted into a separate table which uses mysql partitioning to efficiently truncate rows older than 24 hours.

We will likely want to use partitioning again, which is going to require performance and structural considerations – mainly what are we partitioning over? `(preserve, date_format(updated_at, 'YYMMDD'))` may work but will also increase the size of the primary key.

### 🏃 Import stable scores to new storage

An importer for this purpose has [been created](https://github.com/ppy/osu-queue-score-statistics/blob/master/osu.Server.Queues.ScorePump/ImportHighScores.cs)
. A test import was run on production in December, which took around 5 days (limited by threading of the importer, so can be vastly improved).

This import was to test compatibility mostly. We will need to reinitialise these scores once a linking table has been created, because as it stands there's no way to know which score relates to the original `score_id`s.

### ⏱ Implement and test score wiping (in case we need to remove scores for all lazer or specific lazer, etc.)

We should be able to, given one or more client version hashes, remove all related scores (and rollback relevant stats). This is something we haven't been able to do until now, which has limited the ability to have test runs of game-breaking changes or to limit rollback of bugs to only those scores submitted against the affected version(s).

Make sure that this process can be run efficiently regardless of the size of the wipe.

### ⏱ Balancing, forward plans, etc.

There's a lot to discuss in terms of balancing and integration of the two score sources. Some that come to mind are:

- we need to figure out how the two methods of scoring are going to work together going forward
- smoogipoo wants to discuss the concept of "prestige" leaderboards
- i want to abolish mod multipliers

### Statistics

For simplicity, these stats are *just for the osu! ruleset*. The overall number is around 30-40% higher than this.

24h Scores (stored regardless of pass/fail)
---------

Scores ingested: 9,284,715 Ingested scores which are non-fail: 1,864,121 Storage size: 3.7gb Size per score: 237 bytes

Overall High Scores
---------

Scores stored: 1,517,428,252 Storage size: 230gb Size per score: 167 bytes Delta (24h): ~1,300,000 scores
