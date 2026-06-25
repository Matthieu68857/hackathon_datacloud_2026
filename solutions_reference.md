# 🔑 Disneyland Agentic Data Cloud gHack - Solutions Reference

This document contains the complete reference implementations, configurations, and code blocks for all tracks of the Disneyland Agentic Data Cloud gHack. These solutions should be kept separate from the main attendee-facing instructions.

---

## 📍 Track 1: Data Foundation (Operational to Analytical)

### 1.2 Generate Vector Embeddings at the Source
```sql
-- Enable extensions
CREATE EXTENSION IF NOT EXISTS vector CASCADE;
CREATE EXTENSION IF NOT EXISTS google_ml_integration CASCADE;

-- Add embedding column
ALTER TABLE disneyland_attractions ADD COLUMN embedding vector(3072);

-- Generate embeddings using Vertex AI
UPDATE disneyland_attractions
SET embedding = google_ml.embedding('gemini-embedding-001', description);

-- Checkpoint validation similarity search:
-- Find top 5 attractions similar to 'thrilling dark ride in space'
SELECT name, description, 1 - (embedding <=> google_ml.embedding('gemini-embedding-001', 'thrilling dark ride in space')::vector) AS similarity
FROM disneyland_attractions
ORDER BY embedding <=> google_ml.embedding('gemini-embedding-001', 'thrilling dark ride in space')::vector ASC
LIMIT 5;
```

### 1.3 Logical Replication CDC Configuration (If configured manually)
```sql
CREATE PUBLICATION pub_disney FOR TABLE disneyland_reviews, disneyland_attractions;
ALTER USER postgres WITH REPLICATION;
SELECT PG_CREATE_LOGICAL_REPLICATION_SLOT('slot_disney', 'pgoutput');
```

---

## 📍 Track 2: Exposing AlloyDB via QueryData & MCP Toolbox

### 2.1 Bridge BigQuery Analytics to AlloyDB (BigQuery FDW)
```sql
-- Enable extension and configure FDW server
CREATE EXTENSION IF NOT EXISTS bigquery_fdw;
CREATE SERVER bq_disney_server FOREIGN DATA WRAPPER bigquery_fdw;
CREATE USER MAPPING FOR postgres SERVER bq_disney_server;

-- Map BigQuery forecasted wait times table
CREATE FOREIGN TABLE bq_forecasted_waiting_times (
    attraction_id int,
    forecasted_timestamp timestamp,
    predicted_wait_time numeric
) SERVER bq_disney_server OPTIONS (
    project '[YOUR_PROJECT_ID]',
    dataset 'disney',
    table 'forecasted_waiting_times'
);

-- Map BigQuery graph recommendations table
CREATE FOREIGN TABLE bq_graph_recommendations (
    attraction_id int,
    recommended_next_attraction_id int,
    congestion_level text
) SERVER bq_disney_server OPTIONS (
    project '[YOUR_PROJECT_ID]',
    dataset 'disney',
    table 'graph_recommendations'
);
```

### 2.2 QueryData Context Set (`querydata_disney_context.json`)
```json
{
  "templates": [
    {
      "nlQuery": "Show available attractions in Disneyland Paris",
      "sql": "SELECT name, description FROM public.disneyland_attractions WHERE branch = 'Disneyland Paris'",
      "intent": "List all attractions for a specific park branch",
      "manifest": "List attractions by branch",
      "parameterized": {
        "parameterized_intent": "Show available attractions in $1",
        "parameterized_sql": "SELECT name, description FROM public.disneyland_attractions WHERE branch = $1"
      }
    },
    {
      "nlQuery": "Find reviews with rating 5 for Space Mountain",
      "sql": "SELECT r.review_id, r.rating, r.review_text FROM public.disneyland_reviews r INNER JOIN public.disneyland_attractions a ON r.branch = a.branch WHERE a.name = 'Space Mountain' AND r.rating = 5",
      "intent": "Get reviews with a specific rating for a named attraction",
      "manifest": "Get reviews by attraction and rating",
      "parameterized": {
        "parameterized_intent": "Find reviews with rating $2 for $1",
        "parameterized_sql": "SELECT r.review_id, r.rating, r.review_text FROM public.disneyland_reviews r INNER JOIN public.disneyland_attractions a ON r.branch = a.branch WHERE a.name = $1 AND r.rating = $2"
      }
    },
    {
      "nlQuery": "Average rating of attractions in California Adventure",
      "sql": "SELECT AVG(rating) FROM public.disneyland_reviews WHERE branch = 'California Adventure'",
      "intent": "Calculate the average review rating for a specific branch",
      "manifest": "Average rating by branch",
      "parameterized": {
        "parameterized_intent": "Average rating of attractions in $1",
        "parameterized_sql": "SELECT AVG(rating) FROM public.disneyland_reviews WHERE branch = $1"
      }
    }
  ],
  "facets": [
    {
      "sql_snippet": "r.rating >= 4",
      "intent": "highly rated reviews",
      "manifest": "Filter reviews by a minimum rating threshold",
      "parameterized": {
        "parameterized_intent": "reviews with rating greater than or equal to $1",
        "parameterized_sql_snippet": "r.rating >= $1"
      }
    }
  ],
  "value_searches": [
    {
      "query": "SELECT DISTINCT T.name as value, 'public.disneyland_attractions.name' as columns, 'Attraction Name' as concept_type, 0 as distance, '{}'::text as context FROM public.disneyland_attractions T WHERE T.name ILIKE $value",
      "concept_type": "Attraction Name",
      "description": "Search for attraction names"
    },
    {
      "query": "SELECT DISTINCT T.branch as value, 'public.disneyland_attractions.branch' as columns, 'Branch Name' as concept_type, 0 as distance, '{}'::text as context FROM public.disneyland_attractions T WHERE T.branch ILIKE $value",
      "concept_type": "Branch Name",
      "description": "Search for park branches"
    }
  ]
}
```

### 2.3 Expose AlloyDB AI Operators
```sql
-- Create index for Keyword Search (FTS)
CREATE INDEX IF NOT EXISTS attractions_tsvector_idx ON disneyland_attractions USING RUM (description_tsvector rum_tsvector_ops);

-- Create index for Vector Cosine Similarity (ScaNN)
CREATE INDEX IF NOT EXISTS attractions_vector_idx ON disneyland_attractions USING scann (embedding cosine) WITH (num_leaves=10);

-- Custom PostgreSQL function utilizing AlloyDB AI google_ml.if
CREATE OR REPLACE FUNCTION filter_attractions_semantically(prompt_text TEXT, max_id INT)
RETURNS TABLE(name TEXT, description TEXT) AS $$
  SELECT name, description 
  FROM disneyland_attractions 
  WHERE attraction_id <= max_id
    AND google_ml.if(
      prompt => prompt_text || ' Description: ' || description
    );
$$ LANGUAGE SQL;
```

### 2.4 MCP Toolbox Configuration (`tools.yaml`)
```yaml
kind: source
name: disney-db
type: alloydb-postgres
project: "[YOUR_PROJECT_ID]"
region: "europe-west1"
cluster: "[YOUR_CLUSTER]"
instance: "[YOUR_INSTANCE]"
ipType: "public"
database: "postgres"
user: "postgres"
password: "buildwithgemini2026"
---
# Tool 1: Hybrid Search using ScaNN and Full-Text Search
kind: tool
name: search_attractions_hybrid
type: postgres-sql
source: disney-db
description: "Performs a high-performance hybrid (vector + keyword) search on park attractions based on user interests."
parameters:
  - name: vector_query
    type: string
    description: "Semantic search term (e.g., 'thrilling space roller coaster')"
  - name: text_query
    type: string
    description: "Keyword search term (e.g., 'Space Mountain')"
statement: |
  SET google_ml_integration.enable_preview_ai_functions = true;
  SELECT a.name, a.description, search_results.score
  FROM disneyland_attractions a
  JOIN ai.hybrid_search(
    search_inputs => ARRAY[
        $$ {
          "data_type": "vector",
          "table_name": "disneyland_attractions",
          "key_column": "attraction_id",
          "vec_column": "embedding",
          "distance_operator": "public.<=>",
          "limit": 5,
          "query_vector": "ai.embedding('gemini-embedding-001', $1)::vector"
        } $$::JSONB,
        $$ {
          "data_type": "text",
          "table_name": "disneyland_attractions",
          "key_column": "attraction_id",
          "text_column": "description_tsvector",
          "limit": 5,
          "ranking_function": "<=>",
          "query_text_input": $2
        } $$::JSONB
    ],
    id_type => NULL::BIGINT
  ) AS search_results ON a.attraction_id = search_results.id;
---
# Tool 2: Semantic Filtering using AlloyDB AI operator (ai.if)
kind: tool
name: semantic_ride_filter
type: postgres-sql
source: disney-db
description: "Filters attractions semantically based on a natural language safety or suitability prompt (e.g., 'suitable for pregnant women')."
parameters:
  - name: prompt_text
    type: string
  - name: max_id
    type: integer
statement: |
  SELECT * FROM filter_attractions_semantically($1, $2);
---
# Tool 3: Transactional Tool to record new reviews
kind: tool
name: add_attraction_review
type: postgres-sql
source: disney-db
description: "Saves a new customer review for an attraction into the operational database."
parameters:
  - name: rating
    type: integer
  - name: review_text
    type: string
  - name: branch
    type: string
statement: |
  INSERT INTO public.disneyland_reviews (review_id, rating, review_text, branch, year_month)
  VALUES ((SELECT COALESCE(MAX(review_id), 0) + 1 FROM public.disneyland_reviews), $1, $2, $3, TO_CHAR(CURRENT_DATE, 'YYYY-MM'))
  RETURNING review_id, rating, branch;
---
# Tool 4: Analytical Tool checking Wait Time Forecasts from FDW
kind: tool
name: get_wait_time_forecast
type: postgres-sql
source: disney-db
description: "Queries BigQuery FDW to get forecasted wait times for a specific attraction."
parameters:
  - name: attraction_id
    type: integer
statement: |
  SELECT predicted_wait_time 
  FROM bq_forecasted_waiting_times 
  WHERE attraction_id = $1 
  ORDER BY forecasted_timestamp ASC LIMIT 1;
---
# Tool 5: Analytical Tool checking Graph Recommendations from FDW
kind: tool
name: get_next_ride_recommendation
type: postgres-sql
source: disney-db
description: "Gets graph-based next-ride routing recommendations for a guest leaving a specific attraction to avoid queues."
parameters:
  - name: attraction_id
    type: integer
statement: |
  SELECT recommended_next_attraction_id, congestion_level 
  FROM bq_graph_recommendations 
  WHERE attraction_id = $1;
---
kind: toolset
name: disneyland_operational_tools
tools:
  - search_attractions_hybrid
  - semantic_ride_filter
  - add_attraction_review
  - get_wait_time_forecast
  - get_next_ride_recommendation
```

---

## 📍 Track 7: Building the Real-Time Guest Assistant

### 7.1 ADK Agent Implementation (`agent.py`)
```python
from google.adk.agents import Agent
from toolbox_core import ToolboxSyncClient

# 1. Connect to the MCP server running on port 5000
toolbox = ToolboxSyncClient("http://127.0.0.1:5000")

# 2. Load all tools (operational + analytical)
disney_tools = toolbox.load_toolset('disneyland_operational_tools')

# 3. Define the Guest Guide Agent
visitor_guide = Agent(
    name='disney_guide_agent',
    model="gemini-2.5-flash",
    description='A helpful, friendly guide for Disneyland visitors.',
    instruction="""
    You are a friendly Disneyland Guest Assistant. Your goal is to help visitors plan their day.
    Use the tools at your disposal to:
    - Search for attractions (using hybrid search) or filter them semantically.
    - Check forecasted wait times for attractions.
    - Recommend the next ride to avoid queues based on graph recommendations.
    - Add customer reviews if they want to review a ride.
    
    Always maintain a magical, helpful, and friendly tone.
    """,
    tools=disney_tools,
)
```
