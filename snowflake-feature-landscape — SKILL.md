   
name: snowflake-feature-landscape description: "Show every Snowflake feature being actively used in your account, grouped by product area, with user counts and activity status. Use when: what Snowflake features am I using, feature usage report, what products do we use in Snowflake, how many users on each feature, my Snowflake usage landscape, feature adoption report, show my Snowflake usage. Requires SELECT on SNOWFLAKE.ACCOUNT\_USAGE. No configuration needed — runs on your connected account automatically."

 

# Snowflake Feature Landscape

Shows every Snowflake feature and product area your account is actively using, grouped by product category, with user counts and activity status. Runs entirely on your own Snowflake account data — no external data sources required.

**Data source:** `SNOWFLAKE.ACCOUNT_USAGE` (your Snowflake account's built-in usage schema)

**Coverage:** Cortex AI functions, Cortex Agents, Dynamic Tables, Tasks, Serverless Tasks, Snowpipe, Replication, and core SQL workloads (SELECT, DML, COPY, CALL). Some features detectable only at query-type level — see note in output.

# **Prerequisites**

Your connected role must have access to ACCOUNT\_USAGE:

`-- Run as ACCOUNTADMIN once to grant access:`  
`GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE <your_role>;`  
ACCOUNT\_USAGE data has up to 3-hour latency (QUERY\_HISTORY: \~45 minutes). Very recent activity may not yet appear.

# **Workflow**

## **Step 1: Identify Account**

`SELECT CURRENT_ACCOUNT() AS ACCOUNT_NAME, CURRENT_ORGANIZATION_NAME() AS ORG_NAME`  
Use the result as the report title: "\[ORG\_NAME\] — \[ACCOUNT\_NAME\]"

## **Step 2: Detect Active Features**

Single UNION query across all detectable feature signals. Run as one statement:

`SELECT 'AI/ML'            AS PRODUCT_CATEGORY,`  
       `'Cortex Functions' AS USE_CASE,`  
       `FUNCTION_NAME      AS FEATURE,`  
       `MIN(DATE(START_TIME))              AS FIRST_USE_DATE,`  
       `MAX(DATE(START_TIME))              AS LAST_USE_DATE,`  
       `COUNT(DISTINCT DATE(START_TIME))   AS ACTIVE_DAYS,`  
       `COUNT(DISTINCT USER_NAME)          AS UNIQUE_USERS,`  
       `COUNT(*)                           AS TOTAL_CALLS`  
`FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_FUNCTIONS_USAGE_HISTORY`  
`WHERE START_TIME >= DATEADD('month', -12, CURRENT_TIMESTAMP())`  
`GROUP BY FUNCTION_NAME`

`UNION ALL`

`SELECT 'AI/ML', 'Agents', 'Cortex Agents',`  
       `MIN(DATE(START_TIME)), MAX(DATE(START_TIME)),`  
       `COUNT(DISTINCT DATE(START_TIME)), COUNT(DISTINCT USER_NAME), COUNT(*)`  
`FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AGENT_USAGE_HISTORY`  
`WHERE START_TIME >= DATEADD('month', -12, CURRENT_TIMESTAMP())`

`UNION ALL`

`SELECT 'Data Engineering', 'Transformation', 'Dynamic Tables',`  
       `MIN(DATE(REFRESH_START_TIME)), MAX(DATE(REFRESH_START_TIME)),`  
       `COUNT(DISTINCT DATE(REFRESH_START_TIME)), NULL, COUNT(*)`  
`FROM SNOWFLAKE.ACCOUNT_USAGE.DYNAMIC_TABLE_REFRESH_HISTORY`  
`WHERE REFRESH_START_TIME >= DATEADD('month', -12, CURRENT_TIMESTAMP())`

`UNION ALL`

`SELECT 'Data Engineering', 'Transformation', 'Tasks',`  
       `MIN(DATE(SCHEDULED_TIME)), MAX(DATE(SCHEDULED_TIME)),`  
       `COUNT(DISTINCT DATE(SCHEDULED_TIME)), NULL, COUNT(*)`  
`FROM SNOWFLAKE.ACCOUNT_USAGE.TASK_HISTORY`  
`WHERE SCHEDULED_TIME >= DATEADD('month', -12, CURRENT_TIMESTAMP())`

`UNION ALL`

`SELECT 'Data Engineering', 'Transformation', 'Serverless Tasks',`  
       `MIN(DATE(START_TIME)), MAX(DATE(START_TIME)),`  
       `COUNT(DISTINCT DATE(START_TIME)), NULL, COUNT(*)`  
`FROM SNOWFLAKE.ACCOUNT_USAGE.SERVERLESS_TASK_HISTORY`  
`WHERE START_TIME >= DATEADD('month', -12, CURRENT_TIMESTAMP())`

`UNION ALL`

`SELECT 'Data Engineering', 'Ingestion', 'Snowpipe',`  
       `MIN(DATE(START_TIME)), MAX(DATE(START_TIME)),`  
       `COUNT(DISTINCT DATE(START_TIME)), NULL, COUNT(*)`  
`FROM SNOWFLAKE.ACCOUNT_USAGE.PIPE_USAGE_HISTORY`  
`WHERE START_TIME >= DATEADD('month', -12, CURRENT_TIMESTAMP())`

`UNION ALL`

`SELECT 'Platform', 'Replication', 'Replication',`  
       `MIN(DATE(START_TIME)), MAX(DATE(START_TIME)),`  
       `COUNT(DISTINCT DATE(START_TIME)), NULL, COUNT(*)`  
`FROM SNOWFLAKE.ACCOUNT_USAGE.REPLICATION_USAGE_HISTORY`  
`WHERE START_TIME >= DATEADD('month', -12, CURRENT_TIMESTAMP())`

`UNION ALL`

`SELECT`  
  `CASE`  
    `WHEN QUERY_TYPE = 'SELECT' THEN 'Analytics'`  
    `WHEN QUERY_TYPE = 'CALL'   THEN 'Analytics'`  
    `ELSE 'Data Engineering'`  
  `END AS PRODUCT_CATEGORY,`  
  `CASE`  
    `WHEN QUERY_TYPE = 'SELECT'                           THEN 'Business Intelligence'`  
    `WHEN QUERY_TYPE = 'CALL'                             THEN 'Stored Procedures & UDFs'`  
    `WHEN QUERY_TYPE IN ('INSERT', 'UPDATE', 'DELETE', 'MERGE') THEN 'Transformation'`  
    `WHEN QUERY_TYPE ILIKE 'COPY%'                        THEN 'Ingestion'`  
    `ELSE 'SQL Operations'`  
  `END AS USE_CASE,`  
  `QUERY_TYPE AS FEATURE,`  
  `MIN(DATE(START_TIME)), MAX(DATE(START_TIME)),`  
  `COUNT(DISTINCT DATE(START_TIME)), COUNT(DISTINCT USER_NAME), COUNT(*)`  
`FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY`  
`WHERE START_TIME >= DATEADD('month', -12, CURRENT_TIMESTAMP())`  
  `AND QUERY_TYPE IN ('SELECT', 'INSERT', 'UPDATE', 'DELETE', 'MERGE', 'COPY', 'CALL')`  
`GROUP BY QUERY_TYPE`

## **Step 3: Add Status and Sort**

After getting results, compute STATUS:

* Active \= LAST\_USE\_DATE \>= DATEADD('day', \-30, CURRENT\_DATE())  
* Inactive \= LAST\_USE\_DATE \< DATEADD('day', \-30, CURRENT\_DATE())

Sort order within each PRODUCT\_CATEGORY: Active first, then Inactive; alphabetically by FEATURE within each group.

Render "—" (not 0\) for UNIQUE\_USERS when the value is NULL — this means usage tracking for that feature is account-level rather than user-level, not that zero users touched it.

## **Step 4: Format and Display**

 

# **Snowflake Feature Landscape — \[ORG\_NAME / ACCOUNT\_NAME\]**

*Reporting period: last 12 months | Active \= used within last 30 days | Inactive \= not seen in last 30 days* *Note: Feature detection uses ACCOUNT\_USAGE views. Some Snowflake features (e.g., Streamlit, Snowpark specifically) may be grouped under broader SQL query types rather than listed individually.*

## **\[Product Category 1\]**

| Use Case | Feature | First Used | La\[Product Category 2\] ... Summary: X features active · Y features inactive · Z product categories · N total unique users   Step 5: Offer Google Doc Export Ask: "Would you like me to create a Google Doc version of this to share with your team?" Wait for the user's response before proceeding. If yes — create using Google Workspace MCP tools: Title: `[Account Name] — Snowflake Feature Landscape [Today's Date]` Same structure as the in-chat output Intro paragraph: "This document summarizes the Snowflake features and product areas active in your account over the last 12 months. Features marked as inactive have not been used in the past 30 days." Footer: "Data as of \[today's date\] | Source: SNOWFLAKE.ACCOUNT\_USAGE" Stopping Points Step 5: Wait for Google Doc confirmation Output In-chat: grouped markdown table by product category with user volume and status Optional: Google Doc for team sharing How to Install Option A — GitHub (recommended): /github-plugin-installer https://github.com/\<repo-url\> Option B — Local file: Place this SKILL.md in a folder named `snowflake-feature-landscape`, then in CoCo run /local-plugin-installer and point to the folder when prompted. Option C — Manual: mkdir \-p \~/.snowflake/cortex/skills/snowflake-feature-landscape Copy SKILL.md into that folder, then restart CoCo. Once installed, invoke with /snowflake-feature-landscape or ask "what Snowflake features am I using?" st Used | Active Days | Unique Users | Total Calls | Status |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| ... | ... | ... | ... | ... | ... | ... | Active / Inactive |
