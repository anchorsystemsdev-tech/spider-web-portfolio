 # Creator workflow-v3 with Logging

## Description
This workflow provides a managed interface for creating new n8n workflows via a Webhook. It acts as a wrapper for the n8n Public API, adding a robust logging layer. Every request to create a workflow is intercepted, parsed, and logged into a PostgreSQL database, followed by a second log entry recording the success or failure of the API response.

## Nodes Used
*   **Webhook**: Entry point for POST requests containing workflow definitions.
*   **Code**: 
    *   Extracts payload data.
    *   Formats structured log entries for "Request" and "Response" states.
    *   Executes the n8n API call using a custom JavaScript HTTP request.
*   **PostgreSQL**: Inserts audit trails into `request_logs` and `response_logs` tables.
*   **Respond to Webhook**: Returns the API response and log metadata to the original caller.

## Required Credentials
*   **Postgres account**: Credentials to access the PostgreSQL instance for logging.
*   **n8n Public API Key**: Note that in this version of the workflow, the API key is hardcoded in the "http" Code node header (`X-N8N-API-KEY`).

## How to Use
1.  **Database Setup**: Ensure you have a PostgreSQL database with two tables:
    *   `request_logs`: Columns for `timestamp`, `type`, `operation`, `workflow_name`, `nodes_count`, and `request_body`.
    *   `response_logs`: Columns for `timestamp`, `type`, `status`, `workflow_id`, `workflow_name`, `error`, and `full_response`.
2.  **API Configuration**: Open the **"http"** Code node and update the `url` (if not using localhost) and the `X-N8N-API-KEY` with your actual n8n Public API key.
3.  **Deployment**: Set the workflow to **Active**.
4.  **Triggering**: Send a POST request to the Webhook path `create-workflow-v3`.
    *   **Payload Example**:
        ```json
        {
          "name": "My Automated Workflow",
          "nodes": [...],
          "settings": {}
        }
        ```

## Workflow Structure
1.  **Ingestion**: The **Webhook** receives the workflow JSON.
2.  **Request Logging**: The workflow formats a log entry and immediately inserts it into the `request_logs` table via the **Postgres** node.
3.  **API Interaction**: The **"http"** node sends the payload to the n8n instance's internal API (`/api/v1/workflows`) to physically create the workflow.
4.  **Response Processing**: A JavaScript node calculates whether the creation succeeded (based on the presence of a Workflow ID) and formats a second log entry.
5.  **Response Logging**: The outcome (including error messages if applicable) is saved to the `response_logs` table.
6.  **Finalization**: The **Respond to Webhook** node delivers the result back to the user, confirming the new Workflow ID.