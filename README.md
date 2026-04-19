<img width="926" height="120" alt="image" src="https://github.com/user-attachments/assets/77c6d816-0ff8-44e8-b300-5743ff21a637" /><img width="921" height="219" alt="image" src="https://github.com/user-attachments/assets/d77e49a2-3280-434a-8e20-c3db5d5ea165" />



# auto_compact_daemon
When using graphify using compact costs large amounts of tokens based on the length of the conversation. This is a backdoor fix to the compact issue, instead of using claude to write the compact to graphiffy, this daemon uses Ollama on your local GPU to summarize the exact same text, but for free. Offloading the text in a .md file.


# 🧠 Zero-Cost Local Memory Daemon

A lightweight, $0.00 background RAG (Retrieval-Augmented Generation) poller for agentic CLI tools like Claude Code. 

This daemon watches an export queue, uses a local offline LLM (via Ollama) to summarize your chat history, and seamlessly injects the compressed context back into your project's knowledge map—completely bypassing expensive API compaction taxes.

## ⚠️ The Problem
When using CLI-based AI agents (like Claude Code), your hot memory bloats quickly. Commands like `/compact` send your massive chat history to an expensive frontier API just to write a short summary. This "token tax" adds up rapidly.

## 💡 The Solution
Instead of paying a cloud API to read your chat logs, this daemon uses your local GPU/CPU to do the heavy lifting for free. 
1. You dump your chat memory to a local folder.
2. The background PowerShell daemon detects the file.
3. A local model (like Qwen 2.5 Coder) summarizes the architecture and variables.
4. The script injects the summary into your local project markdown map.
5. You wipe your agent's hot memory and reload the map—costing you $0.00.

---

## 🛠️ Prerequisites & Installation (Windows)

### 1. Install Ollama
Ollama runs the AI models physically on your local machine. Open an Administrator PowerShell window and install it via the Windows Package Manager:
```powershell

winget install Ollama.Ollama

```

Option A: The Visual Way (Click-by-Click)
If you prefer to use standard Windows menus, here is exactly where this setting is hidden:

Open the Start Menu: Press the Windows key on your keyboard.

Search: Type exactly this: Edit environment variables for your account.

Open the Menu: Click the matching result that pops up. (A small window named "Environment Variables" will appear).

Create a New Note: Look at the top half of that window (labeled "User variables for [YourName]"). Click the New... button right below that top box.

Fill in the Blanks: A tiny box will pop up asking for two things. Paste these exactly:

Variable name: OLLAMA_NO_CLOUD

Variable value: 1

Save it: Click OK on that tiny box, and then click OK on the main window to close it.

Restart Ollama: If Ollama was already running, quit it from your system tray (the little llama icon by your clock) and restart it so it reads the new sticky note.

Option B: The 1-Second Shortcut (PowerShell)
Since you are already comfortable copying and pasting commands into PowerShell, you can skip all those menus entirely. Windows allows you to create this sticky note using one line of code.

Open a standard PowerShell window and paste this exact command, then hit Enter:

```PowerShell
[Environment]::SetEnvironmentVariable("OLLAMA_NO_CLOUD", "1", "User")
```
That is it. The command silently creates the exact same variable. Just restart your PowerShell windows and Ollama, and your local AI is permanently air-gapped from the internet.

🚀 Setup & Deployment
1. Project Structure
Navigate to your current project root directory. Your structure should look like this:

```
Plaintext
your_project/
├── export_queue/             # The daemon will create this automatically
├── project_context.md        # Your master context/RAG file
└── auto_compact_daemon.ps1   # The background script
```

2. Create the Daemon Script
Create a file named auto_compact_daemon.ps1 in your project root and paste the following PowerShell code:

```PowerShell
# auto_compact_daemon.ps1 - $0.00 Automated Background RAG Poller (Windows)

$WATCH_DIR = "export_queue"
$GRAPH_FILE = "project_context.md" # Change this to your master context file
$MODEL = "qwen2.5-coder:7b"
$POLL_INTERVAL = 5

# 1. Setup the queue directory
if (-not (Test-Path -Path $WATCH_DIR)) {
    New-Item -ItemType Directory -Path $WATCH_DIR | Out-Null
}

if (-not (Test-Path -Path $GRAPH_FILE)) {
    Write-Host "Error: $GRAPH_FILE not found. Please create your master context markdown file first." -ForegroundColor Red
    exit
}

Write-Host "==========================================================" -ForegroundColor Cyan
Write-Host " 🤖 Auto-Compaction Daemon Active (Windows)" -ForegroundColor Cyan
Write-Host " 📁 Watching directory : .\$WATCH_DIR\" -ForegroundColor Cyan
Write-Host " 🧠 Using Local Model  : $MODEL" -ForegroundColor Cyan
Write-Host " ⏱️  Polling Interval   : $POLL_INTERVAL seconds" -ForegroundColor Cyan
Write-Host "==========================================================" -ForegroundColor Cyan
Write-Host "Usage in Claude Code: /export export_queue\chat1.md`n" -ForegroundColor Yellow

while ($true) {
    # 2. Find the oldest file in the queue (FIFO - First In, First Out)
    $TARGET_FILE = Get-ChildItem -Path $WATCH_DIR -File | Sort-Object CreationTime | Select-Object -First 1

    if ($null -ne $TARGET_FILE) {
        $FULL_PATH = $TARGET_FILE.FullName
        $TIMESTAMP = Get-Date -Format "HH:mm:ss"
        
        Write-Host "[$TIMESTAMP] 📥 Picked up: $($TARGET_FILE.Name) from the queue." -ForegroundColor Yellow
        
        $RAW_CHAT = Get-Content -Path $FULL_PATH -Raw
        
        # Adjust this prompt to fit your specific workflow
        $PROMPT = "You are a strict technical summarizer. Extract the core architectural decisions, variables, script modifications, and methodology from this chat transcript. Write a dense, highly technical markdown summary. Do NOT output conversational filler. Output ONLY the raw markdown summary. Here is the chat: $RAW_CHAT"

        Write-Host "[$TIMESTAMP] ⚙️ Condensing with local Ollama model..." -ForegroundColor DarkGray
        
        # 3. Call Ollama via PIPELINE to bypass Windows command-line length limits
        $SUMMARY = $PROMPT | ollama run $MODEL $PROMPT

        $SUMMARY = $SUMMARY -replace "`e\[[0-9;]*[a-zA-Z]", ""

        Write-Host "[$TIMESTAMP] 💉 Injecting into context map..." -ForegroundColor Green
        
        $INJECT_TIME = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
        $HEADER = "## [LOCAL COMPACTION UPDATE - $INJECT_TIME]"
        
        # 4. Append to the context map cleanly
        Add-Content -Path $GRAPH_FILE -Value ""
        Add-Content -Path $GRAPH_FILE -Value $HEADER
        Add-Content -Path $GRAPH_FILE -Value ""
        Add-Content -Path $GRAPH_FILE -Value $SUMMARY

        # 5. Delete the file to advance the queue
        Remove-Item -Path $FULL_PATH -Force
        
        Write-Host "[$TIMESTAMP] 🗑️ Deleted $($TARGET_FILE.Name). Waiting for next export..." -ForegroundColor Red
        Write-Host "----------------------------------------------------------"
    }

    # 6. Rest loop to prevent CPU thrashing
    Start-Sleep -Seconds $POLL_INTERVAL
}
```

3. Start the Daemon
Open a dedicated PowerShell window that you can leave running in the background. If you haven't enabled script execution on your machine yet, run Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser first.

Then start the poller:

```PowerShell
.\auto_compact_daemon.ps1
```

The Daily Workflow
You stay entirely inside your primary AI terminal (like Claude Code). When your hot memory gets too large and queries start getting expensive, execute this two-step process:

1. Dump your memory to the queue:

```Plaintext
/export export_queue\1.md
```
(The background daemon will instantly detect the file, fire up your local GPU to summarize it, inject it into your project_context.md file, and delete the export file).

2. Wipe the expensive hot memory:

```Plaintext
/clear
```
On your next prompt, simply ask the agent to re-read your project_context.md file. It will instantly absorb the newly appended summary as absolute truth, and your API context window starts fresh at practically $0.00.


## ⚖️ Disclaimer & Liability
This is an experimental, open-source script designed to optimize API costs. Because this daemon operates a queue that automatically **deletes** files once they are processed, you must ensure you are only exporting disposable chat logs into the watched directory. 

By using this tool, you acknowledge that you are using it "AS-IS" and entirely at your own risk. The author assumes absolutely no liability for any lost data, deleted files, corrupted markdown maps, or broken pipelines. **Always keep backups of your master context files.**
