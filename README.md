# Snowflake CoCo Skills for Customers
This repository contains two Cortex Code (CoCo) skills that Snowflake customers can install in their own CoCo Desktop. Once installed, they run entirely on the customer's own Snowflake account data — no internal Snowflake systems, credentials, or configuration required.
# SKILLS IN THIS REPOSITORY
snowflake-feature-landscape Command: /snowflake-feature-landscape Shows every Snowflake feature your account has used in the last 12 months, grouped by product area, with user counts and activity status. Flags features that have gone quiet in the last 30 days.

my-snowflake-radar Command: /my-snowflake-radar Maps recent Snowflake product launches to the features you are actively using and the projects your team is currently building. Fetches live release notes from docs.snowflake.com and uses AI reasoning to surface what is most relevant to your specific stack.

## PREREQUISITES
The following must be in place before installing either skill.
1. CoCo Desktop installed and connected to your Snowflake account Download from: https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code
2. ACCOUNT_USAGE access granted. Your connected role must have SELECT access to the SNOWFLAKE.ACCOUNT_USAGE schema. Run this once as ACCOUNTADMIN: GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE <your_role>;
3. web_search enabled (for my-snowflake-radar only). This is a CoCo setting controlled by your Snowflake admin. If it is disabled, the radar skill degrades gracefully — it still delivers your feature stack analysis but skips the live product launch section.
4. Google Workspace MCP configured (optional). Required only if you want to export results to a Google Doc at the end of a skill run. Set it up by running /google_workspace_install in CoCo.
## HOW TO INSTALL
### Option A — GitHub (recommended)
In your CoCo Desktop chat, run: /github-plugin-installer https://github.com/<your-repo>/snowflake-coco-skills
CoCo clones the repository and installs both skills globally. They appear in the / command picker on your next prompt.
### Option B — Local folder
Download or unzip the skill files from this repository to your computer. Then in CoCo Desktop chat, run: /local-plugin-installer
Point to the folder when prompted.
### Option C — Manual
Open a terminal and run: mkdir -p ~/.snowflake/cortex/skills/snowflake-feature-landscape mkdir -p ~/.snowflake/cortex/skills/my-snowflake-radar
Copy the SKILL.md file from each folder in this repository into the matching directory on your machine. Restart CoCo Desktop. Both skills appear in the / picker automatically.

## SKILL 1: SNOWFLAKE FEATURE LANDSCAPE
Command: /snowflake-feature-landscape Who it is for: Data leaders, platform owners, and anyone who wants a clear picture of which Snowflake features their account is actively using.
What it does
Queries your SNOWFLAKE.ACCOUNT_USAGE schema to detect every Snowflake feature used in the last 12 months. Results are grouped by product category — AI/ML, Analytics, Data Engineering, and Platform — and show the specific feature, first and last use dates, number of active days, unique user count, and total call volume. Features not seen in the last 30 days are flagged as inactive.
At the end of the run, the skill offers to create a Google Doc you can share with your team or your Snowflake account team.
Features detected
Cortex AI functions at function-level granularity (Cortex Analyst, Cortex Search, Cortex Complete, and others) Cortex Agents Dynamic Tables Tasks and Serverless Tasks Snowpipe Replication Core SQL workloads: SELECT, DML (INSERT / UPDATE / DELETE / MERGE), COPY, CALL
Considerations
ACCOUNT_USAGE data has up to 3-hour latency. Very recent activity may not yet appear.
Some features — including Streamlit and Snowpark specifically — are grouped under broader SQL query types because ACCOUNT_USAGE does not expose them as separate event types. The output includes a note explaining this.
No cost or revenue figures are shown. The skill displays only usage counts and dates.
Example output
Snowflake Feature Landscape — YOUR_ORG / YOUR_ACCOUNT Reporting period: last 12 months
AI/ML Cortex Functions — Cortex Analyst: first used Aug 2025, last used Jul 17 2026, 95 active days, 11 unique users, 968 calls, Active Cortex Functions — Cortex Complete: first used Sep 2025, last used Jul 16 2026, 105 active days, — users, — calls, Active
Data Engineering Transformation — Dynamic Tables: first used Jul 2025, last used Jul 17 2026, 268 active days, 62 unique users, 132K calls, Active Ingestion — Snowpipe: first used Jul 2025, last used Jul 17 2026, 363 active days, — users, — calls, Active
Summary: 42 features active · 5 features inactive · 4 product categories · 1,847 total unique users

## SKILL 2: MY SNOWFLAKE RADAR
Command: /my-snowflake-radar Who it is for: Product managers, architects, and technical leaders who want to understand how recent Snowflake announcements relate to what their team is actively using and building.
What it does
Combines four inputs to produce a personalized product radar:
Your active feature stack from ACCOUNT_USAGE (last 90 days) Your overall usage trend from warehouse metering (last 6 months) A brief description of your current Snowflake initiatives that you provide in one to three sentences Recent Snowflake product announcements fetched live from docs.snowflake.com
Using these inputs, the skill applies AI reasoning to classify each recent Snowflake launch into one of three buckets and produces a three-section output.
What's Changing In Your Stack shows launches that directly affect features you are already using.
New Capabilities to Consider shows launches in product areas you are already invested in but where you have not yet adopted this specific capability.
Active Projects That Could Benefit maps your described initiatives to specific Snowflake announcements that may accelerate that work.
At the end of the run, the skill offers to create a Google Doc of the output.
Considerations
Product launch data is fetched live from docs.snowflake.com at the time you run the skill. Private Preview features may not appear in public release notes.
Matching between release note feature names and ACCOUNT_USAGE feature names is done by AI reasoning, not exact string matching. Occasional mismatches are possible.
If web_search is unavailable, the skill delivers the feature stack and trend analysis only and clearly states that live product launch data could not be fetched.
The output contains no internal Snowflake pricing, sales, or account data of any kind. It is entirely safe to share externally.
Example output
My Snowflake Radar — YOUR_ACCOUNT Active stack: last 90 days | Release notes: July 2026
What's Changing In Your Stack Cortex Managed Agents (GA): Simplifies operating agents reliably at scale. Your matching usage: Agents API, Cortex Analyst via Agents. CoCo Desktop (GA): Delivers 2x faster engineering tasks with 51% fewer tokens. Your matching usage: Cortex Code.
New Capabilities to Consider ML Online Feature Store and Model Serving (Public Preview): Collapses two requests per prediction into one. Your related usage: Snowflake ML on WH.
Active Projects That Could Benefit Your initiative: Building an AI assistant for internal users Relevant launch: Cortex Managed Agents How it helps: Makes agent lifecycle management production-ready without custom orchestration code.
REPOSITORY STRUCTURE
snowflake-coco-skills/   snowflake-feature-landscape/     SKILL.md   my-snowflake-radar/     SKILL.md
Both skills are single-file. No dependencies, no scripts, and no additional configuration required beyond the prerequisites above.
FREQUENTLY ASKED QUESTIONS
Do these skills send my data anywhere outside Snowflake? No. The feature landscape skill queries only your own ACCOUNT_USAGE schema — data never leaves your Snowflake account. The radar skill additionally performs a web search on public Snowflake documentation at docs.snowflake.com, which does not transmit any of your account data.
What role do I need? Any role with IMPORTED PRIVILEGES on the SNOWFLAKE database. This is typically granted to SYSADMIN or a custom analytics role by ACCOUNTADMIN. You do not need ACCOUNTADMIN to run the skills.
The skill returned no results. What happened? The most likely cause is that your connected role does not have IMPORTED PRIVILEGES on the SNOWFLAKE database. Run the grant shown in the Prerequisites section and try again.
Can I run both skills in the same CoCo session? Yes. Run /snowflake-feature-landscape first to get a complete 12-month inventory, then run /my-snowflake-radar for the 90-day active stack and product launch matching.
How current is the data? ACCOUNT_USAGE views have up to 3-hour latency. QUERY_HISTORY is closer to 45 minutes. Release notes from docs.snowflake.com are fetched live at the time you run the skill.
Who maintains these skills? These skills are maintained by the Snowflake field team. Contact your Snowflake account team with questions or to request updates.
Published by Snowflake Field Engineering
