---
name: mobile-interstitials-status
description: >-
  Use when a user who has the Metabase MCP connector asks for the status,
  health, or "how things are going" with mobile interstitials — e.g. "what's
  the status on mobile interstitials", "как дела с мобильными интерстишлами",
  "статус по интерстишлам", "проверь монетизацию интерстишлами", the splash →
  banner → purchase funnel, or the winback/regular splash funnel. The skill
  reviews the "Winback Splash Funnel" dashboard across every segment, surfaces
  anything anomalous, and runs follow-up Metabase queries to dig deeper.
  Requires the Metabase MCP connector. Do not use without it.
---

# Mobile Interstitials Status

## 0. Announce activation + language (REQUIRED, do this first)
Before any tool call, the first line of the reply must clearly state that this
skill is running, e.g.:

> **🔧 Skill: Mobile Interstitials Status** — pulling the funnel dashboard and checking every segment.

Never run the skill silently.

**Language:** answer in the same language the user asked in (Russian request →
Russian answer, English → English, etc.). This applies to the activation banner
and the whole report.

## 1. Where the data lives
- **Dashboard:** `Winback Splash Funnel`, id **353** (locate by name if the id differs).
- **Collection:** `Interstitials Splash Funnel` (id 649).
- **Database:** `UG Core`, id **2** — ClickHouse (locate by name if id differs).
- **Table:** `` `default`.`ug_rt_events_app` `` (alias `evt`).
- All cards are read with `execute_card`. Deeper ad-hoc research uses `execute_query` on database 2.
- **Read-only skill.** Never modify, archive, or re-save cards/dashboard. Only read and query.

## 2. The funnel
`Eligible (Tab View)` → `Splash View` → `Banner Upgrade View` → `Banner Purchase Click` → `Purchase Process Finish` (called **Access**).

Every metric is split by **platform** (iOS = `UGT_IOS`, Android = `UGT_ANDROID`) and **Splash type** (Regular / Winback). `uniqExact(unified_id)` is the unit everywhere.

## 3. Card inventory (execute all, both platforms)
Absolute dynamics — stacked bar by Splash type, metric `Users`:
- Splash View: **8849** iOS / **8853** Android
- Banner Upgrade View: **8850** iOS / **8854** Android
- Banner Purchase Click: **8851** iOS / **8855** Android
- Purchase Process Finish (Access): **8852** iOS / **8856** Android

Version mix — stacked bar by Version: **8857** iOS / **8858** Android

Conversions — combo (bar = denominator / "from" step, lines = rate + 7-day moving average MA7):
- splash → banner: **8859** iOS / **8867** Android
- banner → click: **8862** iOS / **8868** Android
- click → access: **8865** iOS / **8869** Android
- splash → access: **8866** iOS / **8870** Android

Pre-step:
- Eligible for splash (stacked bar by type): **8871** iOS / **8872** Android
- eligible → splash (combo): **8873** iOS / **8874** Android

## 4. Procedure
1. State activation + match the user's language (section 0).
2. Pull every card with `execute_card` (`max_rows: -1`). Conversion rates come back as **fractions** (0.36 = 36%); the MA7 column is the 7-day trailing average.
3. For each card build the segment view: iOS vs Android, Regular vs Winback, and the latest complete day vs the trailing ~7-day MA7.
4. Apply the anomaly checklist (section 5).
5. For anything flagged, drill in with `execute_query` (sections 6–7) before reporting.
6. Write the status report (section "Output format").

Note: the dashboard excludes the current day (`date < today()`), default window starts `2026-05-03`. The latest meaningful day is "yesterday".

## 5. What counts as suspicious
- **Any conversion > 100%** (> 1.0 as a fraction). Post-rework this is impossible; if seen, a card regressed (likely lost the session-linking join) — flag it explicitly.
- **Rate breaks the MA7**: latest daily rate deviates materially from its MA7 (rule of thumb > ~20–25% relative), or a sustained multi-day trend.
- **Volume cliff**: day-over-day drop in `Users` at any step beyond normal weekday variation (≈ >25–30%), or a step that goes to zero.
- **Platform divergence**: iOS and Android moving in opposite directions on the same metric.
- **Type divergence**: Winback vs Regular behaving very differently from their usual gap.
- **seq mix shift**: spike in `Unknown` seq_number share, or the seq distribution shifting (use template B).
- **Version anomaly**: a newly dominant version with a much worse conversion, or an old version's share not decaying (use template C).
- **Eligible vs splash**: `eligible → splash` dropping means eligible users stopped seeing splashes (delivery problem upstream of monetization).
- **Empty / stale**: a card returns no rows, or the latest date lags well behind yesterday.
Always describe the segment, the magnitude, and the dates. Don't alarm on normal weekend dips.

## 6. Data model & rules (so ad-hoc queries match the cards)
Fields on `` `default`.`ug_rt_events_app` ``: `event`, `value` (splash source string), `unified_id`, `source` (`UGT_IOS`/`UGT_ANDROID`), `app_version`, `datetime` (UTC), `date`, `session_id`, `rights`, and nested arrays `` `params.key` `` (Array(String)) / `` `params.value` `` (Array(UInt32)).

**Always filter** `` `session_id` > 0 AND `unified_id` > 0 ``, always set an explicit
lower date bound, and always exclude today with `` `date` < today() ``.

**Splash type** (from a Splash View's `value`):
```sql
CASE WHEN `value` LIKE '%Winback%' OR `value` = 'AD Final Interstitial' THEN 'Winback' ELSE 'Regular' END
```
Splash funnel restricts splashes to interstitials: `` `value` LIKE '%Interstitial%' ``.

**seq_number** (shown-splash sequence, from Splash View params; values are numeric, so `toString`):
```sql
if(indexOf(`params.key`, 'seq_number') = 0, 'Unknown',
   toString(`params.value`[indexOf(`params.key`, 'seq_number')]))
```

**Linking post-splash events to their Splash View**: same `unified_id + session_id`, event `datetime >= splash.datetime`. Implement as a per-session `max(datetime)` aggregate of the downstream event, LEFT JOINed to the splash; `has_step = (agg.last_dt >= splash.splash_dt)`. seq_number and Splash type of a post-splash event come from the **linked** Splash View, never from the event's own value.

**Conversions** = `uniqExactIf(unified_id, has_from AND has_to) / uniqExactIf(unified_id, has_from)` (denominator is the "from" step; splash-origin denominators use plain `uniqExact`). MA7 = `avg(rate) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)`.

**Eligible (pre-splash, Tab View + `rights`)**: Regular = `rights % 10 = 0`; Winback = `rights % 10 IN (4,5)`.

## 7. Ready-to-run query templates (`execute_query`, database 2)
These mirror the card logic. They are raw SQL (no Metabase `{{ }}` tags), so the
date bounds are **literals you must set** before running — never drop them.
Swap `source`, the event names, and the date window as needed.

### Template A — daily session-linked conversion for one step (any platform / step)
Set the platform in both CTEs, the "to" event in `step`, and the date window in both CTEs.
```sql
WITH
`splash` AS (
    SELECT
        `evt`.`date`       AS `date`,
        `evt`.`unified_id` AS `unified_id`,
        `evt`.`session_id` AS `session_id`,
        `evt`.`datetime`   AS `splash_dt`,
        CASE WHEN `evt`.`value` LIKE '%Winback%' OR `evt`.`value` = 'AD Final Interstitial'
             THEN 'Winback' ELSE 'Regular' END AS `splash_type`,
        if(indexOf(`params.key`, 'seq_number') = 0, 'Unknown',
           toString(`params.value`[indexOf(`params.key`, 'seq_number')])) AS `seq_number`
    FROM `default`.`ug_rt_events_app` AS `evt`
    WHERE `evt`.`event`   = 'Splash View'
      AND `evt`.`source`  = 'UGT_IOS'              -- << platform
      AND `evt`.`value` LIKE '%Interstitial%'
      AND `evt`.`session_id` > 0
      AND `evt`.`unified_id` > 0
      AND `evt`.`date` >= '2026-05-03'             -- << MANDATORY lower bound
      AND `evt`.`date` <  today()                  -- exclude today
      -- optional: AND `seq_number` = '2'  (paste the full if(...) expression here, not the alias)
),
`step` AS (
    SELECT
        `evt`.`unified_id` AS `unified_id`,
        `evt`.`session_id` AS `session_id`,
        max(`evt`.`datetime`) AS `last_dt`
    FROM `default`.`ug_rt_events_app` AS `evt`
    WHERE `evt`.`event`   = 'Banner Upgrade View'  -- << "to" step (Banner Purchase Click / Purchase Process Finish)
      AND `evt`.`source`  = 'UGT_IOS'              -- << same platform
      AND `evt`.`session_id` > 0
      AND `evt`.`unified_id` > 0
      AND `evt`.`date` >= '2026-05-03'             -- << same window
      AND `evt`.`date` <  today()
    GROUP BY `unified_id`, `session_id`
)
SELECT
    `s`.`date`        AS `date`,
    `s`.`splash_type` AS `splash_type`,
    uniqExact(`s`.`unified_id`)                                            AS `from_users`,
    uniqExactIf(`s`.`unified_id`, `st`.`last_dt` >= `s`.`splash_dt`)       AS `to_users`,
    uniqExactIf(`s`.`unified_id`, `st`.`last_dt` >= `s`.`splash_dt`)
        / nullIf(uniqExact(`s`.`unified_id`), 0)                           AS `conv`
FROM `splash` AS `s`
LEFT JOIN `step` AS `st`
       ON `s`.`unified_id` = `st`.`unified_id`
      AND `s`.`session_id` = `st`.`session_id`
GROUP BY `date`, `splash_type`
ORDER BY `date`, `splash_type`
```

### Template B — drop-off by `seq_number` (where in the splash sequence users fall off)
Splash → Banner reach, bucketed by the shown-splash number. Use a short recent window.
```sql
WITH
`splash` AS (
    SELECT
        `evt`.`unified_id` AS `unified_id`,
        `evt`.`session_id` AS `session_id`,
        `evt`.`datetime`   AS `splash_dt`,
        if(indexOf(`params.key`, 'seq_number') = 0, 'Unknown',
           toString(`params.value`[indexOf(`params.key`, 'seq_number')])) AS `seq_number`
    FROM `default`.`ug_rt_events_app` AS `evt`
    WHERE `evt`.`event`   = 'Splash View'
      AND `evt`.`source`  = 'UGT_IOS'              -- << platform
      AND `evt`.`value` LIKE '%Interstitial%'
      AND `evt`.`session_id` > 0
      AND `evt`.`unified_id` > 0
      AND `evt`.`date` >= '2026-05-28'             -- << MANDATORY window (e.g. last 7 full days)
      AND `evt`.`date` <  today()
),
`banner` AS (
    SELECT
        `evt`.`unified_id` AS `unified_id`,
        `evt`.`session_id` AS `session_id`,
        max(`evt`.`datetime`) AS `last_dt`
    FROM `default`.`ug_rt_events_app` AS `evt`
    WHERE `evt`.`event`   = 'Banner Upgrade View'
      AND `evt`.`source`  = 'UGT_IOS'
      AND `evt`.`session_id` > 0
      AND `evt`.`unified_id` > 0
      AND `evt`.`date` >= '2026-05-28'
      AND `evt`.`date` <  today()
    GROUP BY `unified_id`, `session_id`
)
SELECT
    `s`.`seq_number` AS `seq_number`,
    uniqExact(`s`.`unified_id`)                                        AS `splash_users`,
    uniqExactIf(`s`.`unified_id`, `b`.`last_dt` >= `s`.`splash_dt`)    AS `banner_users`,
    uniqExactIf(`s`.`unified_id`, `b`.`last_dt` >= `s`.`splash_dt`)
        / nullIf(uniqExact(`s`.`unified_id`), 0)                       AS `splash_to_banner`
FROM `splash` AS `s`
LEFT JOIN `banner` AS `b`
       ON `s`.`unified_id` = `b`.`unified_id`
      AND `s`.`session_id` = `b`.`session_id`
GROUP BY `seq_number`
ORDER BY toInt32OrNull(`seq_number`) ASC NULLS LAST   -- numeric order, 'Unknown' last
```

### Template C — release health: splash → access by `app_version`
Spot a bad build. Versions are compared as numeric arrays so `2.10 > 2.9`.
```sql
WITH
`splash` AS (
    SELECT
        `evt`.`unified_id` AS `unified_id`,
        `evt`.`session_id` AS `session_id`,
        `evt`.`datetime`   AS `splash_dt`,
        arrayStringConcat(arrayMap(p -> toString(toUInt32OrZero(p)),
                                   splitByChar('.', `evt`.`app_version`)), '.') AS `version`
    FROM `default`.`ug_rt_events_app` AS `evt`
    WHERE `evt`.`event`   = 'Splash View'
      AND `evt`.`source`  = 'UGT_IOS'              -- << platform
      AND `evt`.`value` LIKE '%Interstitial%'
      AND `evt`.`session_id` > 0
      AND `evt`.`unified_id` > 0
      AND `evt`.`date` >= '2026-05-28'             -- << MANDATORY window
      AND `evt`.`date` <  today()
),
`access` AS (
    SELECT
        `evt`.`unified_id` AS `unified_id`,
        `evt`.`session_id` AS `session_id`,
        max(`evt`.`datetime`) AS `last_dt`
    FROM `default`.`ug_rt_events_app` AS `evt`
    WHERE `evt`.`event`   = 'Purchase Process Finish'
      AND `evt`.`source`  = 'UGT_IOS'
      AND `evt`.`session_id` > 0
      AND `evt`.`unified_id` > 0
      AND `evt`.`date` >= '2026-05-28'
      AND `evt`.`date` <  today()
    GROUP BY `unified_id`, `session_id`
)
SELECT
    `s`.`version` AS `version`,
    uniqExact(`s`.`unified_id`)                                        AS `splash_users`,
    uniqExactIf(`s`.`unified_id`, `a`.`last_dt` >= `s`.`splash_dt`)    AS `access_users`,
    uniqExactIf(`s`.`unified_id`, `a`.`last_dt` >= `s`.`splash_dt`)
        / nullIf(uniqExact(`s`.`unified_id`), 0)                       AS `splash_to_access`
FROM `splash` AS `s`
LEFT JOIN `access` AS `a`
       ON `s`.`unified_id` = `a`.`unified_id`
      AND `s`.`session_id` = `a`.`session_id`
GROUP BY `version`
HAVING `splash_users` >= 500          -- drop long-tail versions
ORDER BY `splash_users` DESC
```

**Template rules:** always keep `session_id > 0`, `unified_id > 0`, an explicit
`date >= '…'` lower bound, and `date < today()`. To filter by splash type add the
`CASE … = 'Winback'` (or `'Regular'`) predicate to the `splash` CTE. To filter by
seq add the full `if(indexOf(...))` expression (not the alias) to the `splash` CTE.
Mid-funnel steps (banner → click, click → access) join **two** downstream
aggregates to the splash and gate the denominator on the "from" step's presence
flag (see section 6).

## Output format
- Line 1: skill-activation banner, in the user's language.
- One-line **overall verdict**: healthy / watch / problem.
- **Funnel snapshot**: latest-day volumes and conversion rates per step, iOS and Android, Regular vs Winback where it matters.
- **Flags**: each anomaly with segment, magnitude, dates, and (if checked) the drill-down result and a hypothesis. If nothing is off, say so plainly.
- Keep it tight; lead with what's wrong. Offer to dig further rather than dumping every number.
