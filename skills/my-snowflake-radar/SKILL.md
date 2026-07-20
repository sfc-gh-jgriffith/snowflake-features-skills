---
name: my-snowflake-radar 
description: "Map recent Snowflake product launches to the features you're actively using and your current projects. Use when: what's new in Snowflake for me, product radar, what Snowflake launches should I care about, what's changing in my Snowflake stack, landscape watch, product changelog for my account, what features are being launched that I use, product updates relevant to me. Requires SELECT on SNOWFLAKE.ACCOUNT\_USAGE and web\_search. No configuration needed — runs on your connected account automatically."
---
 

# My Snowflake Radar

Maps recent Snowflake product launches and feature announcements to what your account is actively using and what your team is currently building. No internal Snowflake data required — runs entirely on your own account usage data and public Snowflake documentation.

**Data sources:**

* `SNOWFLAKE.ACCOUNT_USAGE` — your active feature stack and usage trends  
* `web_search` on docs.snowflake.com — recent Snowflake release notes and feature announcements  
* Your description of current projects (one brief input)

# **Prerequisites**

Your connected role must have access to ACCOUNT\_USAGE:

`-- Run as ACCOUNTADMIN once to grant access:`  
`GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE <your_role>;`  
`web_search` must be available in your CoCo session. If it is disabled, the skill will skip the product launch section and still deliver your feature stack analysis.

# **Workflow**

## **Step 1: Detect Active Feature Stack**

Features used in the last 90 days (active stack only — not historical):

`SELECT 'AI/ML' AS PRODUCT_CATEGORY, 'Cortex Functions' AS USE_CASE,`  
       `FUNCTION_NAME AS FEATURE, MAX(DATE(START_TIME)) AS LAST_USE_DATE,`  
       `COUNT(DISTINCT DATE(START_TIME)) AS ACTIVE_DAYS, COUNT(DISTINCT USER_NAME) AS UNIQUE_USERS`  
`FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_FUNCTIONS_USAGE_HISTORY`  
`WHERE START_TIME >= DATEADD('day', -90, CURRENT_TIMESTAMP())`  
`GROUP BY FUNCTION_NAME`

`UNION ALL`

`SELECT 'AI/ML', 'Agents', 'Cortex Agents',`  
       `MAX(DATE(START_TIME)), COUNT(DISTINCT DATE(START_TIME)), COUNT(DISTINCT USER_NAME)`  
`FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AGENT_USAGE_HISTORY`  
`WHERE START_TIME >= DATEADD('day', -90, CURRENT_TIMESTAMP())`

`UNION ALL`

`SELECT 'Data Engineering', 'Transformation', 'Dynamic Tables',`  
       `MAX(DATE(REFRESH_START_TIME)), COUNT(DISTINCT DATE(REFRESH_START_TIME)), NULL`  
`FROM SNOWFLAKE.ACCOUNT_USAGE.DYNAMIC_TABLE_REFRESH_HISTORY`  
`WHERE REFRESH_START_TIME >= DATEADD('day', -90, CURRENT_TIMESTAMP())`

`UNION ALL`

`SELECT 'Data Engineering', 'Transformation', 'Tasks',`  
       `MAX(DATE(SCHEDULED_TIME)), COUNT(DISTINCT DATE(SCHEDULED_TIME)), NULL`  
`FROM SNOWFLAKE.ACCOUNT_USAGE.TASK_HISTORY`  
`WHERE SCHEDULED_TIME >= DATEADD('day', -90, CURRENT_TIMESTAMP())`

`UNION ALL`

`SELECT 'Data Engineering', 'Transformation', 'Serverless Tasks',`  
       `MAX(DATE(START_TIME)), COUNT(DISTINCT DATE(START_TIME)), NULL`  
`FROM SNOWFLAKE.ACCOUNT_USAGE.SERVERLESS_TASK_HISTORY`  
`WHERE START_TIME >= DATEADD('day', -90, CURRENT_TIMESTAMP())`

`UNION ALL`

`SELECT 'Data Engineering', 'Ingestion', 'Snowpipe',`  
       `MAX(DATE(START_TIME)), COUNT(DISTINCT DATE(START_TIME)), NULL`  
`FROM SNOWFLAKE.ACCOUNT_USAGE.PIPE_USAGE_HISTORY`  
`WHERE START_TIME >= DATEADD('day', -90, CURRENT_TIMESTAMP())`

`UNION ALL`

`SELECT 'Platform', 'Replication', 'Replication',`  
       `MAX(DATE(START_TIME)), COUNT(DISTINCT DATE(START_TIME)), NULL`  
`FROM SNOWFLAKE.ACCOUNT_USAGE.REPLICATION_USAGE_HISTORY`  
`WHERE START_TIME >= DATEADD('day', -90, CURRENT_TIMESTAMP())`

`UNION ALL`

`SELECT`  
  `CASE WHEN QUERY_TYPE = 'SELECT' THEN 'Analytics'`  
       `WHEN QUERY_TYPE = 'CALL'   THEN 'Analytics'`  
       `ELSE 'Data Engineering' END,`  
  `CASE WHEN QUERY_TYPE = 'SELECT' THEN 'Business Intelligence'`  
       `WHEN QUERY_TYPE = 'CALL'   THEN 'Stored Procedures & UDFs'`  
       `WHEN QUERY_TYPE IN ('INSERT','UPDATE','DELETE','MERGE') THEN 'Transformation'`  
       `WHEN QUERY_TYPE ILIKE 'COPY%' THEN 'Ingestion'`  
       `ELSE 'SQL Operations' END,`  
  `QUERY_TYPE,`  
  `MAX(DATE(START_TIME)), COUNT(DISTINCT DATE(START_TIME)), COUNT(DISTINCT USER_NAME)`  
`FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY`  
`WHERE START_TIME >= DATEADD('day', -90, CURRENT_TIMESTAMP())`  
  `AND QUERY_TYPE IN ('SELECT','INSERT','UPDATE','DELETE','MERGE','COPY','CALL')`  
`GROUP BY QUERY_TYPE`

## **Step 2: Compute Usage Trends**

Monthly credit usage over last 6 months to identify what's growing vs. declining:

`SELECT DATE_TRUNC('month', START_TIME) AS MONTH, SUM(CREDITS_USED) AS CREDITS_USED`  
`FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY`  
`WHERE START_TIME >= DATEADD('month', -6, CURRENT_TIMESTAMP())`  
`GROUP BY 1`  
`ORDER BY 1`  
Use LLM to classify overall trend (Growing / Stable / Declining) and note any months with sharp spikes or drops. Also note any features first seen in the last 60 days — these are new adoptions worth highlighting.

## **Step 3: Collect Active Projects**

Ask the user: "Briefly describe your team's current Snowflake initiatives (1–3 sentences). This helps match relevant product launches to your active work."

Use ask\_user\_question (type: "text") with defaultValue "e.g. Building an AI assistant for internal users, migrating our data pipeline, real-time prediction model"

## **Step 4: Fetch Recent Snowflake Release Notes**

Run web\_search with: Snowflake new features release notes \[current year\] site:docs.snowflake.com

Follow the top result URL to extract recent feature announcements. For each, capture: feature name, availability stage (GA / Public Preview / Private Preview), product area, and a brief description.

If web\_search is unavailable or returns no results: inform the user that live release note data is unavailable and proceed to Step 5 with just the feature stack and trends analysis (skip the "What's Changing" and "New Capabilities" sections).

## **Step 5: Match and Format Output**

Use LLM reasoning to cross-reference the active feature stack, trends, and active projects against the fetched release notes. Classify each release feature:

Directly Relevant: Release feature maps to a capability in your active 90-day stack Adjacent Opportunity: Release is in a product area you use, but not this specific feature yet Not Applicable: No usage footprint in this area

When a release feature aligns with a project the user described, note it explicitly.

Output format:

# **My Snowflake Radar — \[CURRENT\_ACCOUNT()\]**

Active stack: last 90 days | Release notes: \[search date\]

## **What's Changing In Your Stack**

Product launches that directly affect features you're already using

| Feature Launch | SNew Capabilities to Consider Launches in product areas you're invested in — worth evaluating Feature Launch SActive Projects That Could Benefit Your initiatives where a recent Snowflake launch may help Your InitiatiStep 6: Offer Google Doc Export Ask: "Would you like me to create a Google Doc of this to share with your team or Snowflake account team?" Wait for the user's response before proceeding. If yes — create using Google Workspace MCP tools: Title: Snowflake Product Radar — \[CURRENT\_ACCOUNT()\] \[Today's Date\] Include all three sections Intro: "This report maps recent Snowflake product announcements to your account's active feature usage and current initiatives." Footer: "Data as of \[today\] | Usage source: SNOWFLAKE.ACCOUNT\_USAGE | Launch data: docs.snowflake.com" Stopping Points Step 3: Wait for project description input Step 6: Wait for Google Doc confirmation Output In-chat: three-section radar fully customer-safe Optional: Google Doc for sharing with team or Snowflake account team How to Install Option A — GitHub (recommended): /github-plugin-installer https://github.com/\<repo-url\> Option B — Local file: Place this SKILL.md in a folder named `my-snowflake-radar`, then in CoCo run /local-plugin-installer and point to the folder when prompted. Option C — Manual: mkdir \-p \~/.snowflake/cortex/skills/my-snowflake-radar Copy SKILL.md into that folder, then restart CoCo. Once installed, invoke with /my-snowflake-radar or ask "what's new in Snowflake for me?" ve Relevant Launch How It Helps \[from user input\] \[feature launch\] \[1-sentence explanation\] tage What It Does Your Related Usage \[feature\] GA/Preview \[description\] \[related features\] tage | Why It Matters to You | Your Matching Features |
| :---- | :---- | :---- | :---- |
| \[feature\] | GA/Preview | \[why it matters\] | \[features from your stack\] |
