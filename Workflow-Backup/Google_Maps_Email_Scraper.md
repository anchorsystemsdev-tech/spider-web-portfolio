 # Google Maps Email Scraper

## Description
This n8n workflow automates the process of lead generation by finding business information on Google Maps and extracting contact emails from their official websites. It takes a search query as input, retrieves local results, scrapes the associated websites, filters out low-quality or system-generated email addresses, and saves unique leads into a Google Sheet while preventing duplicates.

## Nodes Used
- **LangChain Chat Trigger**: Initiates the workflow via a chat interface.
- **Google Sheets**: Used twice—once to read existing leads for deduplication and once to append new leads.
- **HTTP Request**: Used twice—once to query the SerpAPI for Google Maps data and once to scrape individual business websites.
- **Code**: Multiple custom JavaScript nodes handle URL cleaning, email extraction with RegEx, advanced filtering, and data formatting.
- **Split In Batches**: Processes websites in groups of 3 to manage resources and prevent rate-limiting.
- **Wait**: Introduces a 2-second delay between scrapes to ensure polite crawling behavior.

## Required Credentials
- **SerpAPI**: Required for the "SerpAPI Google Maps" node to fetch search results.
- **Google Sheets OAuth2 API**: Required to read from and write to your designated spreadsheet.

## How to Use
1. **Spreadsheet Setup**: Create a Google Sheet with the following headers: `email`, `search_query`, `scraped_at`, `phone`, and `website`.
2. **Credential Configuration**: 
   - Add your SerpAPI key to n8n.
   - Connect your Google account to authorize the Google Sheets nodes.
3. **Node Configuration**:
   - In both Google Sheets nodes ("Read Existing Leads" and "Google Sheets"), select your specific spreadsheet and sheet name from the dropdown.
4. **Execution**:
   - Open the n8n Chat interface or use the execution testing tool.
   - Enter a search query (e.g., "Plumbers in Miami" or "Lawyers in London").
   - The workflow will automatically populate your sheet with unique, filtered leads.

## Workflow Structure
1. **Initialization**: The workflow starts with a search query and immediately reads the Google Sheet to remember existing leads.
2. **Discovery**: It queries Google Maps via SerpAPI. A custom Code node filters the results to extract only primary website URLs, ignoring social media platforms (Facebook, Instagram, etc.).
3. **The Scraping Loop**:
   - URLs are batched (3 at a time).
   - The workflow visits the website and waits 2 seconds.
   - **Extraction & Filtering**: A sophisticated script identifies email addresses but automatically discards:
     - Common spam prefixes (`noreply`, `admin@`, `support@`).
     - Error tracking and system domains (`sentry`, `wixpress`, `bugsnag`).
     - Bot-generated hex hashes (e.g., long strings used by website builders).
4. **Deduplication & Storage**: The workflow compares discovered emails against the list of existing leads. Only new, unique emails are formatted and appended to the Google Sheet.