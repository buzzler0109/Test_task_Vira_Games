Project: Scalable Recursive Google Drive Automation
Platform: n8n (Self-hosted)
Objective: A highly available, asynchronous automation system designed to perform recursive cloning of Google Drive directories, execute dynamic media processing (watermarking), and aggregate real-time statistics without hitting server timeout limits.

1. How to run the workflow (Deployment)
To deploy this system, you must configure its three interconnected micro-workflows:

Import: Import the three provided JSON files into your n8n instance: Main_Orchestrator, Recursive_Worker, and Status_Checker.

Activate Microservices: Open both the Recursive_Worker and Status_Checker workflows. Toggle them to Active (Publish).

Link the Orchestrator: Open the Main_Orchestrator workflow. Locate the Execute Sub-workflow node and Choose "Execute Recursion" from list. Set the Orchestrator to Active.

2. Required Credentials
The following API connections must be configured and authenticated in n8n for the system to function:

Google Drive OAuth2 API: Requires scopes for reading, uploading, modifying, and creating files/folders to perform the recursive cloning and metadata extraction.

Telegram API: Requires a Bot Token generated via BotFather to deliver the final execution report directly to the user or a designated group chat.

3. Workflow Logic & Architecture (How it works)
To ensure high availability and bypass browser timeout limits, the logic is decoupled into a three-tier microservice architecture:

Workflow A: Main Orchestrator (The Manager)

Acts as the entry point. It receives the webhook from the website and immediately returns an asynchronous HTTP 200 OK response containing a unique job_id.

Initializes the destination environment (root folder creation).

Provisions an ephemeral n8n Data Table (copy_stats_{job_id}) to act as a thread-safe local ledger for the current run.

Triggers the Recursive Worker. Upon completion, it reads the ledger, generates a formatted text report, uploads it to Drive, sends a Telegram notification, and deletes the temporary table.

Workflow B: Recursive Worker (The Engine)

Traverses the directory structure. It utilizes a Loop (Split in Batches) pattern to process items sequentially, preventing server Out-of-Memory (OOM) failures.

Folders: Creates a sub-directory and calls itself (recursion).

Images: Downloads the file, dynamically calculates center coordinates/font size based on native resolution, applies the text watermark, and uploads it.

Standard Files: Executes a direct API copy.

Logging: Instantly logs every successful file transfer into the ephemeral Data Table.

Workflow C: Status Checker (Polling Service)

A lightweight, dedicated GET endpoint. While the Worker processes files, the frontend pings this service using the job_id. It queries the Data Table and returns the current file count, enabling a real-time progress bar on the UI.

4. Technical Limitations & Safeguards
Connection Timeouts (504 Gateway Error): Because copying hundreds of files takes minutes, a standard HTTP request will be killed by reverse proxies (like Nginx) or frontend platforms (like Vercel).

Solution Implemented: The asynchronous webhook response combined with the Status_Checker polling mechanism completely circumvents this limitation.

Google Drive API Quotas (429 Too Many Requests): High-speed cloning can trigger rate limits from Google.

Solution Implemented: In the Recursive_Worker all Google Drive nodes are fortified with a Retry On Fail policy (3 attempts, exponential backoff) to handle transient network errors.

Recursion Depth: Maximum directory depth is bound by n8n’s internal limits on nested sub-workflow executions (safely tested up to 15 levels).