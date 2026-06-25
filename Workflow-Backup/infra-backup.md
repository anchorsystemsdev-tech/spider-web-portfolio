 # infra-backup

## Description
This workflow is designed to automate the backup of infrastructure configuration files. It periodically scans a specific directory on the host system (`/your/local/path/`), retrieves the files, and passes them to a secondary workflow named "Localsend-Beam" for transmission to a local network destination (IP: `your-IP`).

## Nodes Used
- **Schedule Trigger**: Triggers the workflow at regular intervals.
- **Read/Write Files from Disk**: Reads files from the filesystem using a recursive glob pattern.
- **Execute Workflow**: Calls a sub-workflow ("Localsend-Beam") to handle the delivery of the backup data.

## Required Credentials
- **Filesystem Access**: No specific n8n "Credentials" object is required, but the n8n instance must have read permissions for the directory `/your/local/path/` on the host machine or within the container.
- **Sub-workflow**: Ensure the workflow `Localsend-Beam` (ID: `your-id`) is present and active in the n8n environment.

## How to Use
1. **Directory Setup**: Ensure your infrastructure files are located in `/your/local/path/`.
2. **Sub-workflow Import**: Import the "Localsend-Beam" workflow into your n8n instance.
3. **Configuration**:
    - Open the **Call 'Localsend-Beam'** node.
    - Update the `ip` parameter (`your-ip`) to match the IP address of your LocalSend receiver.
4. **Schedule**: Adjust the **Schedule Trigger** to your preferred backup frequency (e.g., daily, hourly).
5. **Activation**: Toggle the workflow to **Active**.

## Workflow Structure
1.  **Schedule Trigger**: The entry point that automates the execution based on a time-based interval.
2.  **Read/Write Files from Disk**: Scans the path `/your/local/path/**/*` to grab all configuration and infrastructure files.
3.  **Call 'Localsend-Beam'**: For every file found, it triggers a sub-workflow, passing the target IP and the workflow name (`n8n.yaml`) as inputs to facilitate the transfer.
