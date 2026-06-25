 # Editor workflow-v4

## Description
This workflow provides an automated way to update an existing n8n workflow's configuration via the n8n REST API. It receives a workflow JSON object, sanitizes it by removing read-only system fields (which would otherwise cause API errors during a PUT request), updates the target workflow on the local instance, and then triggers a secondary backup workflow to sync changes to GitHub.

## Nodes Used
*   **Webhook**: Receives the incoming POST request with workflow data.
*   **Code**: Executes JavaScript to extract the workflow ID and strip read-only metadata.
*   **HTTP Request**: Sends the sanitized data to the n8n REST API.
*   **Respond to Webhook**: Returns the API response back to the initial caller.
*   **Execute Workflow**: Triggers a sub-workflow (GitHub Backup) to version control the changes.

## Required Credentials
*   **Header Auth account 2**: An HTTP Header authentication credential used to authorize requests to the n8n REST API (typically requires an `X-N8N-API-KEY`).

## How to Use
1.  **Import the Workflow**: Load this JSON into your n8n instance.
2.  **Configure Credentials**: Ensure "Header Auth account 2" is set up with your n8n API key.
3.  **Local API URL**: The HTTP Request node is configured for `http://localhost:5678`. If your n8n instance uses a different internal URL or port, update this node accordingly.
4.  **Setup Backup Workflow**: Ensure you have a workflow with the ID `your-id` (referred to as "GitHub Backup") to handle the version control part of the process.
5.  **Triggering**: Send a POST request to the `/edit-workflow-v4` webhook path. The body of the request should be the full JSON of the workflow you wish to update.

## Workflow Structure
1.  **Webhook Node**: Entry point. Listens for POST requests.
2.  **Code Node**: 
    *   Extracts the target `id`.
    *   Deletes system-generated fields: `id`, `active`, `createdAt`, `updatedAt`, `versionId`, `meta`, `pinData`, and `staticData`.
    *   Ensures a default `executionOrder` setting is present.
3.  **HTTP Request Node**: Uses a `PUT` method to call `/api/v1/workflows/{id}` on the local n8n instance, passing the updated `name`, `nodes`, `connections`, and `settings`.
4.  **Respond to Webhook Node**: Sends the result of the API call back to the user/service that triggered the workflow.
5.  **Execute Workflow Node**: Calls the "GitHub Backup" workflow asynchronously to save the new workflow state to a repository.