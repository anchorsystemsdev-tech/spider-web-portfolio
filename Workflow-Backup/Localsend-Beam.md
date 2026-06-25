 # Localsend-Beam

## Description
This n8n workflow allows you to programmatically "beam" files to a device running **LocalSend** on your local network. It automates the LocalSend v2 protocol, which involves a "prepare-upload" handshake to exchange session tokens followed by the actual binary data transmission. This is ideal for sending automated backups, reports, or logs directly to a desktop or mobile device without relying on cloud storage.

## Nodes Used
- **Execute Workflow Trigger**: Enables this workflow to be called as a sub-process by other workflows, receiving both JSON metadata and binary file data.
- **HTTP Request (handshake)**: Sends file metadata (name, size, type) to the LocalSend receiver to initiate the session.
- **Merge**: Combines the session tokens received from the handshake with the original binary data from the trigger.
- **HTTP Request (Transfer)**: Performs the final POST request to upload the actual binary content using the authorized session token.

## Required Credentials
No official n8n credentials are required for this workflow. However, it relies on the following environment setup:
- **Network Access**: The n8n instance must have a direct network route to the target device's IP address (defaulted to `your-IP`) on port `1234`.
- **SSL Bypass**: LocalSend typically uses self-signed certificates for local HTTPS. The HTTP nodes are configured to **Ignore TLS Issues** (`allowUnauthorizedCerts: true`) to facilitate this.

## How to Use
1. **Prepare Receiver**: Open LocalSend on your target device (phone, laptop, etc.) and ensure it is connected to the same network.
2. **Configure IP**: Update the URL in both the **handshake** and **Transfer** nodes to match the IP address of your receiving device.
3. **Execute**: Trigger this workflow from another "Parent" workflow. 
   - **Input JSON**: Optionally pass a `name` field (e.g., `My_Backup`). If omitted, it defaults to `backup.json`.
   - **Input Binary**: Provide a binary file on the input named `data`.
4. **Accept Transfer**: Depending on your LocalSend settings (Quick Save), you may need to click "Accept" on the receiving device.

## Workflow Structure
1. **Trigger**: Receives the signal and the binary file to be sent.
2. **Handshake**: The workflow contacts the LocalSend API at `/api/localsend/v2/prepare-upload`. 
   - It calculates the exact file size in bytes using an expression that converts n8n's human-readable strings (like "10 kB" or "2 MB") into integers.
   - It defines the sender as `n8n-cluster-bot`.
3. **Merge**: Because the handshake response contains the necessary `sessionId` and `token` but loses the binary data from the trigger, this node merges them back together.
4. **Transfer**: The workflow sends the binary data to `/api/localsend/v2/upload`. It dynamically injects the `sessionId` and the unique file `token` into the query parameters to authorize the transfer.