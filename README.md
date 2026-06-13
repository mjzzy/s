# Mizz — Complete Documentation

This README fully documents the public API, internal helpers, constants, fields, and files in this repository. It covers behaviors, parameters, return values, example usage, build details, and how to control assembly metadata.

Target framework: .NET 8 (net8.0)

WARNING: The code interacts with system processes and performs injection-like behavior. Only use where legal and permitted.

---

CONTENTS
- Overview
- Public API (detailed)
- Public types
- Private helpers and internal functions
- Fields, constants, and regexes
- Error handling and threading notes
- Files created/modified by recent refactor
- Build and assembly metadata
- Examples
- Changelog of repo edits

---

Overview
The primary entrypoint is the static Api class at SapiV/SapiV/Class1.cs. It wraps an instance of VelAPI (VelocityAPI.dll) to manage attachment to Roblox processes and execute Lua scripts. All members are static and the class targets Windows only via [SupportedOSPlatform("windows")].

---

Public types

- ClientInfo (struct)
  - int Id: Process id for the client (pid)
  - string Name: Process name (e.g., "RobloxPlayerBeta")
  - string Version: Executor/version string (default "Unknown")
  - int State: Numeric state (reserved for client state codes)

---

Public API (detailed per-method)

- bool EnableConsole { get; set; }
  - Default: true
  - Controls whether internal logging uses AllocConsole/FreeConsole and writes to stdout. Turn off to suppress console activity.

- Task Inject()
  - Purpose: Attach to a running Roblox process (RobloxPlayerBeta), ensure Velocity runtime files exist (download/update if necessary), write an AutoExec/CustomNotification.lua file and execute it through Velocity.
  - Behavior:
	1. Checks for running Roblox processes. If none, sets internal _isInjected=false and returns.
	2. Runs FileManager() to ensure dependent files are present; this may start velApi.StartCommunication() which downloads files into a Bin folder.
	3. Writes AutoExec/CustomNotification.lua with the configured LuaScriptNotification payload.
	4. Calls velApi.Attach(pid) and waits for the status result.
	5. On success (VelocityStates.Attached): marks internal _isInjected=true, executes the notification via velApi.Execute, then attempts to delete the temporary file after a short delay.
	6. On failure: sets _isInjected=false, stops communication and cleans up the temporary file if present.
  - Parameters: None
  - Returns: Task that completes once the injection flow finishes (or fails).
  - Exceptions: Errors during attach/execute are caught internally; the method will log failures to console (if enabled) and swallow exceptions after cleanup.

- bool IsInjected()
  - Purpose: Query if the API considers itself injected.
  - Returns: true if internal _isInjected is true and velApi.VelocityStatus == VelocityStates.Attached.

- bool IsRobloxOpen()
  - Purpose: Quick check to see if any RobloxPlayerBeta process exists.
  - Returns: true if any matching process is running.

- void KillRoblox()
  - Purpose: Forcefully kill all RobloxPlayerBeta processes and stop Velocity communication.
  - Behavior: iterates Process.GetProcessesByName("RobloxPlayerBeta") and calls Kill() inside try/catch. Errors are logged to console if enabled.
  - Note: Killing processes may require elevated permissions.

- void Execute(string scriptSource)
  - Purpose: Execute a Lua script in the attached target(s) using the Velocity API.
  - Behavior:
	1. If velApi.injected_pids is empty, returns without action.
	2. Sanitizes the provided script with AntiKick to replace Kick calls.
	3. Prepends optional LuaScriptUserAgent and LuaScriptNameExecutor if configured.
	4. Calls velApi.Execute(fullScript).
	5. On exception, logs the error by allocating a console and printing a message; logging runs in Task.Run to avoid blocking the caller.
  - Parameters:
	- string scriptSource: the raw Lua code supplied by caller.
  - Returns: void (non-blocking on errors).

- void SetCustomInjectionNotification(string title, string text, string idIcon, string duration)
  - Purpose: Configure the notification script used in AutoExec/CustomNotification.lua when injecting.
  - Parameters:
	- title: notification title (defaults to "MIZZ X Api" when null/empty)
	- text: notification text (defaults to "Injected Successfully!")
	- idIcon: Roblox asset id string (defaults to empty)
	- duration: seconds as string (defaults to "5")

- void SetCustomUserAgent(string Name)
  - Purpose: Build Lua code to override request options and set a custom User-Agent header.
  - Parameter: Name (defaults to "MIZZ X Api")
  - The constructed script is stored in LuaScriptUserAgent and prepended to user scripts on Execute.

- void SetCustomNameExecutor(string Name, string Version)
  - Purpose: Provide a Lua snippet that exposes identifyexecutor, getexecutorname and getexploitname. Useful for scripts that query the executor identity.
  - Defaults: Name="MIZZ X Api", Version="v1.0.0"

- List<ClientInfo> GetClientsList()
  - Purpose: Enumerate velApi.injected_pids and return a ClientInfo for each running pid.
  - Behavior: Uses IsProcessRunning(pid) and Process.GetProcessById(pid) to fill Name and Id. Version set to "Unknown".

---

Private helpers and internal functions (documented)

- Task<bool> CheckUpdate()
  - Purpose: Compare local current_version.txt under Bin with remote version at https://getvelocity.lol/assets/current_version.txt to determine whether an update is required.
  - Returns: true if an update is needed or an error occurred (caller uses this conservatively to trigger updates).
  - Exceptions: Exceptions are caught and logged to Output; method returns true on errors.

- bool CheckFiles()
  - Purpose: Verify existence of required directories and files (AutoExec, Bin, Scripts, Workspace, Bin/Decompiler.exe, Bin/injector filename, Bin/current_version.txt).
  - Returns: true when files are missing (note inverted semantic: true => missing files).

- bool IsProcessRunning(int pid)
  - Purpose: Fast check whether a given pid corresponds to a running process.
  - Returns: true if the process exists; false otherwise. Does not allocate console on failure.

- void OnProcessExit(object? sender, EventArgs e)
  - Purpose: Registered with AppDomain.CurrentDomain.ProcessExit to stop velApi communication and remove AutoExec/CustomNotification.lua if present.
  - Behavior: velApi.StopCommunication() is called; file deletion is wrapped in try/catch to avoid throwing during shutdown.

- string AntiKick(string source)
  - Purpose: Replace Kick-style patterns with a print statement to prevent scripts from kicking the client. Uses a compiled Regex KickRegex.

- Task FileManager()
  - Purpose: Ensure the workspace contains the required Velocity distribution files. If files are missing, or if CheckUpdate reports an update, this method will call velApi.StartCommunication() to download files and wait (with timeout) for CheckFiles/CheckUpdate to report success.
  - Behavior: When EnableConsole is true, allocates a console and prints status messages including an ASCII banner and warnings about antivirus.
  - Timeouts: waits up to 120 seconds while polling (1s interval) for files to appear after starting download.

---

Fields, constants, regexes

- private static readonly VelAPI velApi = new VelAPI();
  - Wrapper instance for interacting with Velocity.
- private static bool _isInjected;
  - Tracks injection state locally.
- private static string? LuaScriptNotification, LuaScriptUserAgent, LuaScriptNameExecutor;
  - Optional Lua snippets concatenated onto executed scripts.
- private static string? Velocity_Version;
  - Cached remote velocity version when CheckUpdate runs.
- private static readonly Regex KickRegex
  - Compiled regex that attempts to detect various Kick function usages and patterns commonly used to kick clients; used by AntiKick.
- private const string Msg
  - ASCII banner and a link printed during file download/update console flows.

---

Error handling and threading notes

- Most external operations (process attach, file IO, network) are wrapped in try/catch. The library logs to console when EnableConsole is true.
- Long waits and polling use async Task.Delay to prevent thread pool blocking.
- Execute logs exceptions on a background Task to avoid blocking callers; it does not rethrow.

---

Files created / modified during refactor

- SapiV/SapiV/Class1.cs — main Api class (modified)
- SapiV/SapiV.csproj — project file (modified to include Properties folder and GenerateAssemblyInfo=false)
- SapiV/SapiV/Properties/AssemblyInfo.cs — added to provide one set of assembly attributes (Title/Product/Company set to "MIZZ X")
- README.md — this file was added/expanded to fully document the code base.

---

Build and assembly metadata

- The project targets net8.0. The Velocity API dependency is referenced as a hint path: VelocityAPI.dll (included in repo root and project folder).
- Assembly attributes are provided explicitly in Properties/AssemblyInfo.cs. The project sets <GenerateAssemblyInfo>false</GenerateAssemblyInfo> to avoid duplicate attribute generation.

Changing the produced DLL filename
- To change the compiled DLL name, add an AssemblyName element to the project (SapiV/SapiV.csproj):

```xml
<PropertyGroup>
  <AssemblyName>MIZZ X</AssemblyName>
</PropertyGroup>
```

- After changing project metadata, run dotnet clean and then dotnet build to remove previously-generated obj files and avoid duplicate-attribute issues.

---

Examples

- Basic injection flow (async):

```csharp
Api.SetCustomInjectionNotification("MIZZ X Api", "Injected Successfully!", "", "5");
Api.SetCustomUserAgent("MIZZ X Api");
Api.SetCustomNameExecutor("MIZZ X", "v1.0.0");
await Api.Inject();
```

- Execute script after injection:

```csharp
string lua = "print('Hello from MIZZ X')";
Api.Execute(lua);
```

- Enumerate clients:

```csharp
var clients = Api.GetClientsList();
foreach (var c in clients) Console.WriteLine($"Client {c.Id} {c.Name}");
```

---

If you need XML documentation comments added directly into the source (/// <summary> ... ), or you want the README exported as formatted HTML or a developer reference, I can add those next.

