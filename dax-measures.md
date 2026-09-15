# DAX Measures — IPL Full Analytics Dashboard

This file documents the key DAX measures used across the dashboard, organized by the stage/feature they belong to.

---

## Basic Stage — Core KPIs

```dax
// Total number of matches played
Total Matches = COUNTROWS(ipl_matches_data)

// Total number of distinct IPL seasons in the dataset
Total Seasons = DISTINCTCOUNT(ipl_matches_data[season])

// Total number of distinct cities that hosted matches
Total Cities = DISTINCTCOUNT(ipl_matches_data[city])
```

---

## Advanced Stage — Time Intelligence & Iterators

```dax
// Total runs scored across all deliveries
Total Runs = SUM(ball_by_ball_data[batter_runs])

// Total sixes hit (deliveries where batter_runs = 6)
Total Sixes = 
CALCULATE(
    COUNTROWS(ball_by_ball_data),
    ball_by_ball_data[batter_runs] = 6
)

// Strike Rate leaderboard measure — uses SUMX to calculate accurately per batter
// rather than averaging pre-aggregated values
Strike Rate = 
DIVIDE(
    SUM(ball_by_ball_data[batter_runs]),
    COUNTROWS(ball_by_ball_data)
) * 100
```

---

## Expert Stage — Smart Profile Card (Role-Based Dynamic Logic)

```dax
// Balls faced by the currently selected player (as a batter)
// Wrapped in ALL() to prevent unintended filter leakage from other slicers/highlights
Balls Faced = 
VAR SelPlayer = SELECTEDVALUE('players-data-updated'[player_name])
RETURN
IF(
    ISBLANK(SelPlayer),
    BLANK(),
    CALCULATE(
        COUNTROWS(ball_by_ball_data),
        FILTER(ALL(ball_by_ball_data), ball_by_ball_data[batter] = SelPlayer)
    )
)

// Balls bowled by the currently selected player (as a bowler)
Balls Bowled = 
VAR SelPlayer = SELECTEDVALUE('players-data-updated'[player_name])
RETURN
IF(
    ISBLANK(SelPlayer),
    BLANK(),
    CALCULATE(
        COUNTROWS(ball_by_ball_data),
        FILTER(ALL(ball_by_ball_data), ball_by_ball_data[bowler] = SelPlayer)
    )
)

// Total career runs for the selected player
Total Runs (Player) = 
CALCULATE(
    SUM(ball_by_ball_data[batter_runs]),
    FILTER(
        ALL(ball_by_ball_data),
        ball_by_ball_data[batter] = SELECTEDVALUE('players-data-updated'[player_name])
    )
)

// Total wickets taken by the selected player
// Uses UPPER(... & "") to safely compare is_wicket regardless of its underlying data type
Total Wickets (Player) = 
VAR SelPlayer = SELECTEDVALUE('players-data-updated'[player_name])
RETURN
IF(
    ISBLANK(SelPlayer),
    BLANK(),
    CALCULATE(
        COUNTROWS(ball_by_ball_data),
        FILTER(
            ALL(ball_by_ball_data),
            ball_by_ball_data[bowler] = SelPlayer && UPPER(ball_by_ball_data[is_wicket] & "") = "TRUE"
        )
    )
)

// Runs conceded by the selected player while bowling
Runs Conceded (Player) = 
CALCULATE(
    SUM(ball_by_ball_data[total_runs]),
    FILTER(
        ALL(ball_by_ball_data),
        ball_by_ball_data[bowler] = SELECTEDVALUE('players-data-updated'[player_name])
    )
)

// Economy rate = runs conceded per over bowled
Economy Rate (Player) = 
VAR RunsConceded = [Runs Conceded (Player)]
VAR BallsBowledCount = [Balls Bowled]
VAR OversBowled = DIVIDE(BallsBowledCount, 6)
RETURN
IF(OversBowled > 0, DIVIDE(RunsConceded, OversBowled), BLANK())

// Automatically classifies the selected player's role based on their actual
// batting/bowling volume and wicket-taking record — no manual tagging required
Player Role = 
VAR BF = [Balls Faced]
VAR BB = [Balls Bowled]
VAR WK = [Total Wickets (Player)]
RETURN
SWITCH(
    TRUE(),
    BF > 500 && BB > 300 && WK > 15, "All-Rounder",
    BB > BF, "Bowler",
    BF > 0, "Batter",
    "No Data"
)

// Dynamically labels which stat to show based on the detected role
Dynamic Stat 1 Label = 
SWITCH(
    [Player Role],
    "Batter", "Total Runs",
    "Bowler", "Total Wickets",
    "All-Rounder", "Runs / Wickets",
    "No Data"
)

// Dynamically shows the correct value based on the detected role
Dynamic Stat 1 Value = 
SWITCH(
    [Player Role],
    "Batter", FORMAT([Total Runs (Player)], "#,##0"),
    "Bowler", FORMAT([Total Wickets (Player)], "#,##0"),
    "All-Rounder", FORMAT([Total Runs (Player)], "#,##0") & " / " & FORMAT([Total Wickets (Player)], "#,##0"),
    "—"
)

// Combines label + value into one clean display string for the card
Stat 1 Display = [Dynamic Stat 1 Label] & ": " & [Dynamic Stat 1 Value]
```

---

## Expert Stage — Team Card

```dax
// Total wins for the selected team, explicitly cleared of any pre-existing filter
Team Wins = 
VAR SelTeam = SELECTEDVALUE(teams_data[team_name])
RETURN
IF(
    ISBLANK(SelTeam),
    BLANK(),
    CALCULATE(
        COUNTROWS(ipl_matches_data),
        FILTER(ALL(ipl_matches_data), ipl_matches_data[match_winner] = SelTeam)
    )
)

// Total matches played by the selected team (as team1 OR team2)
Team Matches Played = 
VAR SelTeam = SELECTEDVALUE(teams_data[team_name])
RETURN
IF(
    ISBLANK(SelTeam),
    BLANK(),
    CALCULATE(
        COUNTROWS(ipl_matches_data),
        FILTER(
            ALL(ipl_matches_data),
            ipl_matches_data[team1] = SelTeam || ipl_matches_data[team2] = SelTeam
        )
    )
)

// Losses = matches played minus wins
Team Losses = 
VAR W = [Team Wins]
VAR P = [Team Matches Played]
RETURN
IF(ISBLANK(W) || ISBLANK(P), BLANK(), P - W)

// Total sixes hit by the selected team across all matches
Team Total Sixes = 
VAR SelTeam = SELECTEDVALUE(teams_data[team_name])
RETURN
IF(
    ISBLANK(SelTeam),
    BLANK(),
    CALCULATE(
        COUNTROWS(ball_by_ball_data),
        FILTER(
            ALL(ball_by_ball_data),
            ball_by_ball_data[team_batting] = SelTeam && ball_by_ball_data[batter_runs] = 6
        )
    )
)

// Finds the highest run-scorer for the selected team
Team Top Scorer = 
VAR SelTeam = SELECTEDVALUE(teams_data[team_name])
VAR BatterRuns = 
    SUMMARIZE(
        FILTER(ALL(ball_by_ball_data), ball_by_ball_data[team_batting] = SelTeam),
        ball_by_ball_data[batter],
        "@Runs", SUM(ball_by_ball_data[batter_runs])
    )
RETURN
IF(
    ISBLANK(SelTeam),
    BLANK(),
    MAXX(TOPN(1, BatterRuns, [@Runs]), ball_by_ball_data[batter])
)
```

---

## Key Debugging Lesson

Several early versions of these measures used `FILTER(TableName, condition)` **without** wrapping the table in `ALL()`. This caused values to be silently affected by whatever slicer or cross-filter was already active on the report page — producing numbers that looked plausible but were incorrect (e.g., a player's total runs showing a partial-season figure instead of their full career total).

**Fix:** Always wrap the table in `ALL(TableName)` inside `FILTER()` when the measure is meant to calculate from the complete, unfiltered dataset based on a single explicit condition (like a slicer selection), rather than inheriting the page's existing filter context.

```dax
// Before (bug-prone):
FILTER(ball_by_ball_data, ball_by_ball_data[batter] = SelPlayer)

// After (correct):
FILTER(ALL(ball_by_ball_data), ball_by_ball_data[batter] = SelPlayer)
```

This was validated by cross-checking dashboard outputs against real player statistics until every measure matched reality.
