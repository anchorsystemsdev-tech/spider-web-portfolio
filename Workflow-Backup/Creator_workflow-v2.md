 # Creator workflow-v2

## Description
This workflow provides a programmatic interface to create new n8n workflows. It acts as an API bridge that receives a workflow JSON definition via a Webhook and uses the n8n REST API to install that workflow on the local instance.

## Nodes Used
- **Webhook**: Listens for incoming POST requests.
- **Code in JavaScript**: Extracts the JSON body from the incoming request.
- **HTTP Request**: Communicates with the n8n Public API to create the workflow.
- **Respond to Webhook**: Returns the API response (success or error) to the initial caller.

## Required Credentials
- **Header Auth account 2 (HTTP Header Auth)**: This is required for the **HTTP Request** node to authenticate with the n8n Public API. It should typically include an `X-N8N-API-KEY` header with your personal API key.

## How to Use
1. **Import the Workflow**: Import the JSON into your n8n instance.
2. **Setup Credentials**: Configure the `Your Header Auth account ` with a valid n8n API key.
3. **Configure URL**: By default, the **HTTP Request** node points to `http://your-domain/api/v1/workflows`. If your n8n instance is hosted elsewhere or uses a different port, update this URL.
4. **Activate**: Set the workflow to "Active."
5. **Trigger**: Send a POST request to the webhook URL (ending in `/webhook/create-workflow-v2`). The body of your request should be the valid JSON of the workflow you wish to create.

## Workflow Structure
1. **Incoming Request**: The **Webhook** node receives a POST request containing the workflow data you want to create.
2. **Data Extraction**: The **Code in JavaScript** node runs a small script to isolate the `body` of the webhook request, ensuring only the workflow JSON is passed forward.
3. **API Call**: The **HTTP Request** node sends that JSON to n8n's internal API endpoint for workflow creation.
4. **Final Response**: The **Respond to Webhook** node captures the result from the API (including the new workflow's ID) and sends it back to the original requester as the HTTP response.