<img width="926" height="120" alt="image" src="https://github.com/user-attachments/assets/77c6d816-0ff8-44e8-b300-5743ff21a637" /><img width="921" height="219" alt="image" src="https://github.com/user-attachments/assets/d77e49a2-3280-434a-8e20-c3db5d5ea165" />



# auto_compact_daemon
When using graphify using compact costs large amounts of tokens based on the length of the conversation. This is a backdoor fix to the compact issue, instead of using claude to write the compact to graphiffy, this daemon uses Ollama on your local GPU to summarize the exact same text, but for free. Offloading the text in a .md file.


#  Zero-Cost Local Memory Daemon

A lightweight, $0.00 background RAG (Retrieval-Augmented Generation) poller for agentic CLI tools like Claude Code. 

This daemon watches an export queue, uses a local offline LLM (via Ollama) to summarize your chat history, and seamlessly injects the compressed context back into your project's knowledge map—completely bypassing expensive API compaction taxes.

##  The Problem
When using CLI-based AI agents (like Claude Code), your hot memory bloats quickly. Commands like `/compact` send your massive chat history to an expensive frontier API just to write a short summary. This "token tax" adds up rapidly.

##  The Solution
Instead of paying a cloud API to read your chat logs, this daemon uses your local GPU/CPU to do the heavy lifting for free. 
1. You dump your chat memory to a local folder.
2. The background PowerShell daemon detects the file.
3. A local model (like Qwen 2.5 Coder) summarizes the architecture and variables.
4. The script injects the summary into your local project markdown map.
5. You wipe your agent's hot memory and reload the map—costing you $0.00.

---

##  Prerequisites & Installation (Windows)

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

$WATCH_DIR = "eq"
$GRAPH_FILE = "graphify-out\GRAPH_REPORT.md"
$MODEL = "qwen2.5-coder:7b"
$POLL_INTERVAL = 5

if (-not (Test-Path -Path $WATCH_DIR)) { New-Item -ItemType Directory -Path $WATCH_DIR | Out-Null }

Write-Host "==========================================================" -ForegroundColor Cyan
Write-Host "  Auto-Compaction Daemon Active (Two-Stage Map-Reduce)" -ForegroundColor Cyan
Write-Host "  Watching directory : .\$WATCH_DIR\" -ForegroundColor Cyan
Write-Host "  Using Local Model  : $MODEL" -ForegroundColor Cyan
Write-Host "==========================================================" -ForegroundColor Cyan

while ($true) {
    $TARGET_FILE = Get-ChildItem -Path $WATCH_DIR -File | Sort-Object CreationTime | Select-Object -First 1

    if ($TARGET_FILE) {
        $FULL_PATH = $TARGET_FILE.FullName
        $TIMESTAMP = Get-Date -Format "HH:mm:ss"
        Write-Host "[$TIMESTAMP]  Picked up: $($TARGET_FILE.Name)" -ForegroundColor Yellow
        
        $RAW_CHAT = Get-Content -Path $FULL_PATH -Raw -Encoding UTF8
        
        # 1. Clean terminal artifacts (Preserves Math/Greek symbols)
        $evaluator = [System.Text.RegularExpressions.MatchEvaluator] { param($m); $text=$m.Groups[1].Value; $del=[int]$m.Groups[2].Value; if($text.Length -ge $del){return $text.Substring(0,$text.Length-$del)}; return "" }
        $RAW_CHAT = [System.Text.RegularExpressions.Regex]::Replace($RAW_CHAT, "(?s)(.*?)\x1b\[(\d+)D\x1b\[K\r?\n?", $evaluator)
        $RAW_CHAT = $RAW_CHAT -replace "\x1B\[[0-9;]*[a-zA-Z]", ""

        # 2. Chunking
        $lines = $RAW_CHAT -split "`n"
        if ($lines.Count -lt 3) { $chunks = @( ($lines -join "`n") ) } 
        else {
            $chunkSize = [math]::Ceiling($lines.Count / 3)
            $chunks = @(
                ($lines[0..($chunkSize-1)] -join "`n"),
                ($lines[$chunkSize..($chunkSize*2-1)] -join "`n"),
                ($lines[($chunkSize*2)..($lines.Count-1)] -join "`n")
            )
        }

       # -----------------------------------------------------------------
        # NEW STAGE 1: NATIVE VARIABLE EXTRACTION (Bypass the LLM's Math)
        # -----------------------------------------------------------------
        Write-Host "[$TIMESTAMP]  STAGE 1: Extracting Variables via Regex..." -ForegroundColor DarkGray
        
        # This Regex perfectly captures any line containing a number followed by your exact units
        $VAR_REGEX = '(?im)^.*(?:=|≈)\s*\d+(?:\.\d+)?\s*(?:µs|ms|s|%|GB/s|MB|bytes|lines/second|accesses).*$'
        $NATIVE_VARS = [regex]::Matches($RAW_CHAT, $VAR_REGEX) | ForEach-Object { $_.Value.Trim() } | Select-Object -Unique
        
        $FORMATTED_VARS = "* VARIABLES:`n"
        foreach ($var in $NATIVE_VARS) {
            # Strip leading bullets/dashes from the raw text for clean formatting
            $cleanVar = $var -replace "^[-\*\s]+", ""
            $FORMATTED_VARS += "  - $cleanVar`n"
        }

        # -----------------------------------------------------------------
        # STAGE 2: MAP-REDUCE FOR TOPOLOGY ONLY (Entities & Flows)
        # -----------------------------------------------------------------
        $lines = $RAW_CHAT -split "`n"
        if ($lines.Count -lt 3) { $chunks = @( ($lines -join "`n") ) } 
        else {
            $chunkSize = [math]::Ceiling($lines.Count / 3)
            $chunks = @(
                ($lines[0..($chunkSize-1)] -join "`n"),
                ($lines[$chunkSize..($chunkSize*2-1)] -join "`n"),
                ($lines[($chunkSize*2)..($lines.Count-1)] -join "`n")
            )
        }

        Write-Host "[$TIMESTAMP]  STAGE 2: LLM Extracting Entities and Flows..." -ForegroundColor DarkGray
        $MAPPED_DATA = ""
        
        foreach ($chunk in $chunks) {
            if ([string]::IsNullOrWhiteSpace($chunk)) { continue }
            $PROMPT_MAP = @"
You are an architectural extraction engine. Extract ONLY the architectural entities and processes from the following text.
DO NOT extract numerical variables. DO NOT perform any math. 
--- TEXT TO EXTRACT ---
$chunk
"@
            $BODY = @{ model=$MODEL; prompt=$PROMPT_MAP; stream=$false; options=@{ temperature=0.0; top_p=0.1 } } | ConvertTo-Json -Depth 10 -Compress
            $RESPONSE = Invoke-RestMethod -Uri "http://localhost:11434/api/generate" -Method Post -Body $BODY -ContentType "application/json; charset=utf-8"
            $MAPPED_DATA += "`n" + $RESPONSE.response.Trim()
        }

       Write-Host "[$TIMESTAMP]  STAGE 3: REDUCE (Formatting Graph)..." -ForegroundColor Magenta

        $PROMPT_REDUCE = @"
You are a Graph Topology Formatter. Merge these notes into a single topological map.
STRICT RULES:
1. Use EXACTLY these two headings: * ENTITIES: and * FLOWS:
2. DO NOT output a * VARIABLES: section.
3. FLOW FORMAT: [Entity A] -> [Action] -> [Entity B]
--- NOTES ---
$MAPPED_DATA
"@
        $BODY = @{ model=$MODEL; prompt=$PROMPT_REDUCE; stream=$false; options=@{ temperature=0.0; repeat_penalty=1.2 } } | ConvertTo-Json -Depth 10 -Compress
        $RESPONSE = Invoke-RestMethod -Uri "http://localhost:11434/api/generate" -Method Post -Body $BODY -ContentType "application/json; charset=utf-8"
        $LLM_SUMMARY = $RESPONSE.response.Trim() -replace "`e\[[0-9;]*[a-zA-Z]", "" -replace "\x1b\[[0-9;]*[a-zA-Z]", ""

        # ---------------------------------------------------------
        # THE GUILLOTINE: Force LLM compliance by chopping off its variables
        # ---------------------------------------------------------
        if ($LLM_SUMMARY -match "\* VARIABLES:") {
            $LLM_SUMMARY = ($LLM_SUMMARY -split "\* VARIABLES:")[0].Trim()
        }

        # Clean the Regex variables (Remove accidental 'fio' bash script captures)
        $CLEAN_VARS = "* VARIABLES:`n"
        foreach ($var in $NATIVE_VARS) {
            $cleanVar = $var -replace "^[-\*\s]+", ""
            # Ignore lines from the fio block or orphaned equals signs
            if ($cleanVar -notmatch "(direct=|sync=|bs=|iodepth=|^=\s*[\d\.]+$)") {
                $CLEAN_VARS += "  - $cleanVar`n"
            }
        }

        # Combine LLM Topology with PowerShell's flawless Math extraction
        $FINAL_SUMMARY = "$LLM_SUMMARY`n`n$CLEAN_VARS"

        Write-Host "[$TIMESTAMP]  Injecting into Graphify map..." -ForegroundColor Green
        $INJECT_TIME = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
        $HEADER = "`n`n## [LOCAL COMPACTION UPDATE - $INJECT_TIME]`n"
        
        Add-Content -Path $GRAPH_FILE -Value $HEADER -Encoding UTF8
        Add-Content -Path $GRAPH_FILE -Value $FINAL_SUMMARY -Encoding UTF8
        Remove-Item -Path $FULL_PATH -Force
        
        Write-Host "[$TIMESTAMP]  Deleted $($TARGET_FILE.Name). Waiting..." -ForegroundColor Red
        Write-Host "----------------------------------------------------------"
    }
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


##  Disclaimer & Liability
This is an experimental, open-source script designed to optimize API costs. Because this daemon operates a queue that automatically **deletes** files once they are processed, you must ensure you are only exporting disposable chat logs into the watched directory. 

By using this tool, you acknowledge that you are using it "AS-IS" and entirely at your own risk. The author assumes absolutely no liability for any lost data, deleted files, corrupted markdown maps, or broken pipelines. **Always keep backups of your master context files.**
