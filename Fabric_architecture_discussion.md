When deciding how to summarize massive datasets—such as migrating years of legacy SAS data or analyzing high-volume distribution feeder logs—the architecture you choose dictates both your monthly Azure/Power Platform spend and the real-time accuracy of your Copilot.
Here is a breakdown of the two architectural paths for generating record-level and executive summaries using Copilot Studio.
## Path 1: The Dataverse Dataflow Architecture (Physical Movement)
In this approach, you use ETL processes to physically move data from Microsoft Fabric into Dataverse. Copilot Studio then queries the local Dataverse tables to generate its summaries.
**The Workflow:**
 1. **Extraction:** A Power Apps Dataflow runs on a schedule (e.g., nightly) connecting to your Fabric Warehouse's SQL Analytics Endpoint.
 2. **Loading:** The dataflow physically copies the records (e.g., previous day's NERC compliance logs or feeder outages) into standard Dataverse tables.
 3. **Record-Level Summary:** A user asks the Copilot for information on a specific asset. The Copilot uses the native Dataverse connector to query the rows in Dataverse and summarizes the findings in natural language.
 4. **Executive Summary:** For broader questions ("Summarize system-wide feeder faults for Q3"), Copilot pulls the aggregated rows from Dataverse to build the executive brief.
**Pros:**
 * **Native Integration:** Copilot Studio has incredibly tight, out-of-the-box integration with Dataverse.
 * **Low Latency for Small Datasets:** Once data is in Dataverse, conversational queries are extremely fast.
**Cons:**
 * **Storage Costs:** You are paying premium Dataverse storage rates for data that already exists in OneLake.
 * **Stale Data:** Because Dataflows run on a schedule, the Copilot cannot summarize events that happened between refresh cycles.
## Path 2: The Power Automate Architecture (Live Query & Aggregation)
Instead of moving millions of rows, you use Fabric's powerful compute engine to do the heavy aggregation, passing only the final summarized data back to the Copilot.
**The Workflow:**
 1. **Trigger:** A user asks the Copilot, "Give me an executive summary of substation voltage anomalies from last week."
 2. **Action Invocation:** Copilot Studio identifies the user's intent, extracts the parameters ("voltage anomalies", "last week"), and triggers a connected Power Automate Cloud Flow.
 3. **Live Query Execution:** The Power Automate flow uses the **SQL Server Connector** to pass a dynamically generated SQL query to your Fabric Warehouse.
 4. **Fabric Aggregation:** Fabric executes the SQL query, performing the record-level aggregation at the database level.
 5. **Response:** Power Automate returns the aggregated dataset (in JSON) back to Copilot Studio, which translates the raw metrics into a conversational executive summary.
**Pros:**
 * **Zero Storage Cost:** No data is stored in Dataverse; it remains entirely in OneLake.
 * **Compute Efficiency:** You leverage your existing Fabric F-SKU to handle the heavy SQL aggregation rather than relying on the Copilot LLM to piece together thousands of individual records.
 * **Real-Time Accuracy:** The Copilot has live access to the absolute latest data written to the warehouse.
**Cons:**
 * **Custom Development:** Requires building and maintaining Power Automate flows for the specific actions you want the Copilot to perform.
 * **Latency on Massive Queries:** If the SQL query takes 15 seconds to run in Fabric, the user will wait 15 seconds for the Copilot to respond.
### The Verdict
For an enterprise migrating heavy workloads from systems like SAS or IBM SPSS, **Path 2 (Power Automate) is almost always the superior choice.**
Passing thousands of record-level rows to an LLM context window to generate an executive summary is slow, expensive, and prone to token limits. By using Power Automate to let Fabric execute the SQL GROUP BY and SUM operations natively, you give Copilot Studio exactly what it needs—the aggregated answers—while keeping your Dataverse footprint lean.
