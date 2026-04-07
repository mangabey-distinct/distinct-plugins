# DISTINCT - An AI-enabled data system

## Overview

This skill is part of a distribution of skills, commands, MCP-tools and other capabilities that allow AI agents to interact with a centralized data analysis layer named "Distinct".

The core of the Distinct system is a centralized server that contains information relevant to any data-worker or agent assisting non-tech stakeholders. The centralized servers have the following capabilities and functionality:

- Stores the warehouse tables with metadata like description, table grain, major dimensions, row count, main date columns, suggested interaction limitations (TABLESAMPLE), etc.
- Stores table columns with metadata like sample/most common columns, share of null values, value uniqueness.
- Stores knowledge: Curated bits of valuable information about the data that is not available just by observing the schema. This can be join relationships, caveats in the data, ways in which to interpret the data, shared metric definitions.
- The distinct server also acts as middleware to the data warehouse that makes sure only the tables intended for access by specific end users are actually allowed to be accessed. It checks safety and non-damage of the queries too.
- The distinct server has the concept of dashboards that can be created by AI agents in order to visualize the data for users and these dashboards can be shared internally in the company.

## How to interact with distinct

All the interactions with the distinct server should be done through the `Distinct` MCP server. The name of the server differ based on each installation of the Distinct server. Use these tools in order to gain access to the functionality and knowledge of the Distinct server. The tools have knowledge of your identity and will respect your access rights to the data warehouse.

## The user experience

While you and the user share many parts of the conversation and the user can see most of what you experience through a thought process, not everything is well presented to the user and easy to see/understand. Here are some general notes on the way to have the user experience of your conversation fit into the Distinct ecosystem.

- **Rewrite and present important results/responses**: While the MCP infrastructure is efficient for you, sometimes key results are of interest to the user. That can be in the form of an extra interesting table returned from a query or a table/column description that is returned from another MCP tool, or a key metric definition from searching the knowledge. Do not forget to clearly state what you found. You might even be able to format it as a table or another nice UI element. Do that to keep the user up to speed with the most key logical steps of the analysis.
- **Provide links to dashboards**: These do not show in the current AI client so you should always provide clear links to i.e. newly created dashboards.
