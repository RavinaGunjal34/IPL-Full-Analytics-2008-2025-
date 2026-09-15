# IPL-Full-Analytics-2008-2025

About This Project

Built an end-to-end, interactive Power BI dashboard analyzing 18 IPL seasons (2008–2025) across 1,169 matches and 38 cities, using four raw datasets combined into one relational data model. The project progresses from foundational data cleaning to advanced dynamic DAX logic, including a player card that automatically detects batter vs. bowler role from real match data.

Key Highlights

🔗 Built a relational data model connecting teams, players, matches, and ball-by-ball data — including resolving active/inactive relationship conflicts

📈 Used Time Intelligence and Iterator functions (SUMX) to calculate season-over-season trends and accurate Strike Rate metrics

🎯 Designed a dynamic role-based player card that automatically shows batting or bowling stats depending on who's selected — no manual tagging, purely DAX-driven

🧭 Built drill-through navigation with dedicated Team and Player "Deep Dive" pages

🐞 Identified and fixed a filter-context bug causing a measure to return values inflated by 20x — solved by validating outputs against real player statistics

🎨 Fully custom-branded theme with styled navigation, KPI cards, and consistent design across all pages
