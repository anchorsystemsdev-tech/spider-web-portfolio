 # GitHub Backup

## Description
This workflow automates the process of backing up and documenting your n8n workflows. It fetches all workflows from a local n8n instance, filters for active ones, and uses an AI Agent (Google Gemini) to automatically generate a professional `README.md` for each workflow. The results are then committed to a versioned branch in a specified GitHub repository. Finally, it converts the workflow to a file format and triggers a secondary distribution workflow.

## Nodes Used
- **n8n-nodes-base.scheduleTrigger**: Fires the workflow daily at a set time.
- **n8n-nodes-base.executeWorkflowTrigger**: Allows manual or external workflow execution.
- **n8n-nodes-base.github**: Used to check repository existence and commit JSON/Markdown files.
- **n8n-nodes-base.if**: Validates if the repository exists before proceeding.
- **n8n-nodes-base.httpRequest**: Interacts with the n8n API to fetch workflows and the GitHub Git API to manage branches/refs.
- **n8n-nodes-base.splitOut**: Extracts workflow data from the API response.
- **n8n-nodes-base.code**: Handles JavaScript logic for filtering active workflows, managing version tags, and unpacking data for loops.
- **n8n-nodes-base.splitInBatches**: Iterates through the list of workflows.
- **@n8n/n8n-nodes-langchain.agent**: AI Agent that analyzes workflow JSON to write documentation.
- **@n8n/n8n-nodes-langchain.lmChatGoogleGemini**: The language model (Gemini 1.5 Flash) used for documentation.
- **@n8n/n8n-nodes-langchain.memoryBufferWindow**: Provides context memory for the AI.
- **n8n-nodes-base.convertToFile**: Converts workflow data into a binary JSON file.
- **n8n-nodes-base.executeWorkflow**: Calls the sub-workflow "Localsend-Beam".

## Required Credentials
- **GitHub account (githubApi)**: Required for repository access and committing files.
- **Header Auth (httpHeaderAuth)**: Required for authenticating with the local n8n API (API Key).
- **Google Gemini(PaLM) Api (googlePalmApi)**: Required for AI-generated documentation.

## How to Use
1. **GitHub Setup**: Create a repository named `n8n-workflows-backup` (or update the nodes to reflect your repository name).
2. **n8n API**: Enable the n8n API in your instance settings and create a Personal API Key.
3. **Credentials**: Configure the GitHub, Header Auth (using your n8n API Key), and Google Gemini credentials in your n8n instance.
4. **Endpoint**: If your n8n instance is not hosted at `http://localhost:5678`, update the URL in the "Get All Workflows" node.
5. **Sub-workflow**: Ensure you have a workflow named "Localsend-Beam" or disable that specific node if not needed.
6. **Activation**: Activate the workflow. It will run automatically at 10:00 AM daily.

## Workflow Structure
1. **Trigger**: Starts via a daily schedule or external call.
2. **Pre-check**: Verifies that the GitHub repository is accessible.
3. **Extraction**: Calls the local n8n API, retrieves all workflows, and filters out inactive ones using a Code node.
4. **Git Preparation**: Generates a version tag (e.g., `v12-test-06`) and creates a new branch on GitHub based on the current `main` branch SHA.
5. **Processing Loop**: For every active workflow:
    - An **AI Agent** receives the workflow JSON and generates a `README.md`.
    - The **GitHub node** commits the workflow JSON to the new branch.
    - A second **GitHub node** commits the AI-generated README to the same branch.
    - The workflow is converted to a file and passed to the **Localsend-Beam** sub-workflow.
6. **Completion**: The loop continues until all active workflows are backed up and documented.