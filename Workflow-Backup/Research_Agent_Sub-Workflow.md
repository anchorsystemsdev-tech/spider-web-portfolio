 # Research Agent Sub-Workflow

## Description
This workflow is a specialized sub-workflow designed to perform automated web research and synthesis. It takes a search query, retrieves top results from Google, filters for quality sources, scrapes the content, and utilizes a local LLM (via Ollama) to generate a structured research summary. It is optimized to strip out "noise" such as navigation links, ads, and social media clutter to provide a clean synthesis of information.

## Nodes Used
*   **Execute Workflow Trigger**: Receives the research query from a parent workflow.
*   **SerpAPI (Google Search)**: Performs the initial web search.
*   **Code**: Used for multiple logic steps including URL extraction, domain filtering, text cleaning (noise removal), and final data formatting.
*   **Split In Batches**: Iterates through the discovered URLs.
*   **HTTP Request**: Fetches the raw HTML content of the target websites.
*   **HTML**: Extracts specific article and main content tags from the raw HTML.
*   **Aggregate**: Recombines the processed text from multiple websites into a single dataset.
*   **AI Agent (LangChain)**: The core synthesis engine that follows a specific system prompt.
*   **Ollama Chat Model (LangChain)**: Connects to a local instance of `qwen2.5:3b` to perform the reasoning.

## Required Credentials
*   **SerpAPI**: Needed for the "Google search" node to access organic search results.
*   **Ollama API**: Needed for the "Mistral Model" node to interact with your local AI instance.

## How to Use
1.  **Setup Credentials**: Provide your SerpAPI key and ensure your Ollama instance is accessible (defaulting to the `qwen2.5:3b` model).
2.  **Integration**: Use an **Execute Workflow** node in a parent workflow to call this sub-workflow.
3.  **Input**: Pass a JSON object containing a `query` key (e.g., `{"query": "Future of renewable energy 2025"}`).
4.  **Output**: The workflow returns a structured Markdown summary containing a Topic Summary, Key Points, Details, and Action Items.

## Workflow Structure
1.  **Initiation**: The workflow starts when triggered with a search `query`.
2.  **Discovery**: It searches Google and extracts the top 10 organic links.
3.  **Filtering**: A domain filter removes results from sites like Reddit, Quora, and Medium to prioritize primary articles over social discussions.
4.  **Extraction Loop**:
    *   Fetches HTML for each link.
    *   Extracts text from `<article>` or `<main>` tags.
    *   Cleans the text by removing markdown links, image credits, and social "noise."
5.  **Aggregation**: All cleaned text is joined and truncated to approximately 2,000 characters to ensure it fits within the model's context window.
6.  **Synthesis**: The AI Agent reads the combined content and formats it into a professional research report based on a strict system prompt template.