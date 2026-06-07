# Debug Backend JavaScript in VS Code

PayOS can expose backend JavaScript executed by GraalVM Polyglot through the Chrome DevTools Protocol. This lets application developers debug external app-bundle scripts from VS Code while PayOS runs from a runtime JAR.

This guide documents the verified flow for debugging an app-bundle handler such as `app1/api/orders/GetOrderHandler.js` through a PayOS runtime launched from an app-bundles folder.

For Linux/Bash commands, use [debug-backend-javascript-vscode-linux.md](debug-backend-javascript-vscode-linux.md).

## Requirements

- Java 21 available in the terminal used to launch PayOS.
- A PayOS runtime JAR that includes `org.graalvm.tools:chromeinspector-tool` and the `payos.debug.js` support.
- VS Code with the built-in JavaScript debugger enabled. See [Enable The VS Code JavaScript Debugger](#enable-the-vs-code-javascript-debugger).
- The PayOS application bundle opened or included in VS Code, for example `app1`.
- The backend JavaScript file available on disk, for example `app1/api/orders/GetOrderHandler.js`.

Use a freshly built shaded runtime artifact for `java -jar` launches. Do not debug with an old copied JAR from an app-bundles folder.

## Enable The VS Code JavaScript Debugger

VS Code ships with the JavaScript debugger as a built-in extension. It is usually enabled by default, but it can be disabled globally or for a workspace.

To verify or enable it:

1. Open VS Code.
2. Open the Extensions view with `Ctrl+Shift+X`.
3. Search for `@builtin JavaScript Debugger`.
4. Select **JavaScript Debugger** from Microsoft. The extension identifier is `ms-vscode.js-debug`.
5. If the button says **Enable**, click it. If it says **Disable**, the debugger is already enabled.
6. If VS Code asks to reload the window, reload it before starting the attach configuration.

You do not need the legacy `Debugger for Chrome` extension.

## Prepare The Runtime JAR

If PayOS is launched from an app-bundles folder, rebuild the runtime and copy the freshly built JAR there before starting PayOS.

Example tested layout:

```text
PayOS/
  payos-runtime/
    target/payos-runtime-1.3.0-RELEASE.jar
S2M/payos-bundles/
  bootstrap.json
  payos-runtime-1.3.0-RELEASE.jar
  app1/
    api/orders/GetOrderHandler.js
```

Build and copy the runtime:

```powershell
cd "C:\Projets\DTS\MedTech@Work\Innovation\PayOS\payos-runtime"
mvn clean package

Copy-Item `
  "C:\Projets\DTS\MedTech@Work\Innovation\PayOS\payos-runtime\target\payos-runtime-1.3.0-RELEASE.jar" `
  "C:\Projets\DTS\MedTech@Work\Innovation\S2M\payos-bundles\payos-runtime-1.3.0-RELEASE.jar" `
  -Force
```

The copied JAR must be the same version that contains the JavaScript debug support. If an older runtime JAR is launched, PayOS can execute JavaScript normally but port `9229` will never open.

## Configure VS Code Attach

### Step 1 — Add the app-bundle folder to the VS Code workspace

VS Code only reads `launch.json` from folders that are **workspace roots**. If your app-bundle folder is not already a root in the workspace, the configuration will not appear in Run and Debug.

1. In VS Code, go to **File › Add Folder to Workspace…**
2. Select the root of your app-bundle folder (e.g. `app1/` or its parent `payos-bundles/`).
3. VS Code will reload and the folder will appear in the Explorer as a new workspace root.

> **Tip:** You can save the resulting multi-root workspace as a `.code-workspace` file (**File › Save Workspace As…**) so VS Code reopens all folders automatically next time.

### Step 2 — Create `.vscode/launch.json`

Create the launch file **inside the workspace root you just added**, not in a sub-folder:

```text
app1/.vscode/launch.json          ← correct when app1/ is a workspace root
payos-bundles/.vscode/launch.json ← correct when payos-bundles/ is the workspace root
```

Use `type: "node"`. GraalVM exposes the debug target as a Node-style Chrome DevTools Protocol target, not a browser target.

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Attach PayOS Backend JS",
      "type": "node",
      "request": "attach",
      "address": "127.0.0.1",
      "port": 9229,
      "sourceMaps": false,
      "timeout": 30000
    }
  ]
}
```

After saving the file, open **Run and Debug** (`Ctrl+Shift+D`). The **Attach PayOS Backend JS** entry should now appear in the configuration dropdown at the top of the panel. If it does not appear immediately, reload the VS Code window once (`Ctrl+Shift+P` → **Developer: Reload Window**).

## Launch PayOS With JS Debugging

Start PayOS from the app-bundles folder so relative configuration files such as `bootstrap.json` resolve correctly.

```powershell
cd "C:\Projets\DTS\MedTech@Work\Innovation\S2M\payos-bundles"

java `
  -Dpayos.debug.js=true `
  -Dpayos.debug.js.host=127.0.0.1 `
  -Dpayos.debug.js.port=9229 `
  -Dpayos.debug.js.suspend=true `
  -jar payos-runtime-1.3.0-RELEASE.jar
```

Equivalent environment variables are also supported, but they must be set in the same terminal before starting the JVM:

```powershell
$env:PAYOS_DEBUG_JS = "true"
$env:PAYOS_DEBUG_JS_HOST = "127.0.0.1"
$env:PAYOS_DEBUG_JS_PORT = "9229"
$env:PAYOS_DEBUG_JS_SUSPEND = "true"
java -jar payos-runtime-1.3.0-RELEASE.jar
```

`payos.debug.js.suspend=true` is recommended for VS Code debugging. GraalVM opens the inspector endpoint when the JavaScript context is created. In PayOS, that typically happens when an API request reaches a backend JavaScript handler.

## Trigger The JavaScript Handler

Use an API route that executes the JavaScript file where you set the breakpoint.

For the tested `app1` bundle, `bootstrap.json` exposes PayOS HTTP on `127.0.0.1:8081`, and `app1/config/mappings.json` maps `orders/GetOrderHandler.js` to:

```text
GET /app1/api/orders/{OrderId}
```

Trigger the handler from a second terminal:

```powershell
Invoke-WebRequest `
  -UseBasicParsing `
  http://127.0.0.1:8081/app1/api/orders/123 `
  -Headers @{ `
    'X-Tenant-Id' = 'default'; `
    'X-Correlation-Id' = 'debug-js-test' `
  }
```

With `payos.debug.js.suspend=true`, this request should wait while GraalVM opens the inspector port. That waiting request is expected.

## Verify The Inspector Port

In another terminal, verify that the inspector is open:

```powershell
Invoke-WebRequest -UseBasicParsing http://127.0.0.1:9229/json/list |
  Select-Object -ExpandProperty Content
```

A working response looks like this:

```json
[
  {
    "description": "GraalVM",
    "title": "GraalVM",
    "type": "node",
    "webSocketDebuggerUrl": "ws://127.0.0.1:9229/..."
  }
]
```

The important value is `"type": "node"`; this is why the VS Code attach configuration must also use `"type": "node"`.

You can also verify the listening port directly:

```powershell
Get-NetTCPConnection -LocalAddress 127.0.0.1 -LocalPort 9229 -ErrorAction SilentlyContinue
```

## Attach From VS Code

1. Open the real app-bundle file in VS Code, for example `app1/api/orders/GetOrderHandler.js`.
2. Set a breakpoint in the handler, hook, or library file.
3. Start PayOS with `payos.debug.js=true` and `payos.debug.js.suspend=true`.
4. Call the API route that executes the JavaScript file.
5. Wait until the request is suspended and `http://127.0.0.1:9229/json/list` responds.
6. In VS Code, open **Run and Debug**.
7. Select **Attach PayOS Backend JS** from the app-bundle workspace folder, for example `app1`.
8. Start the attach configuration.
9. If VS Code stops before your breakpoint, click **Continue**.

The breakpoint may not look fully bound before attach. The reliable signs are that the VS Code debug toolbar appears, the suspended HTTP request is controlled by VS Code, and execution stops when it reaches the breakpoint.

If VS Code opens a `debug:` virtual source tab, keep breakpoints in the real file under the app bundle as well. Breakpoint binding depends on the real file URI PayOS passes to GraalVM.

## Supported Source Mapping

PayOS passes real `file:///...` source URIs to GraalVM for:

- API handlers loaded from `api/**/*.js`
- request hooks loaded from `hooks/**/*.js`
- libraries loaded through `$Library`

Because those URIs point to the actual files in the app bundle, VS Code can bind breakpoints directly in the files you own.

## Troubleshooting

### Port 9229 Is Closed

If PowerShell returns `Impossible de se connecter au serveur distant` / `Unable to connect to the remote server`, nothing is listening on `127.0.0.1:9229` yet.

Check these points:

- PayOS is running.
- PayOS was started with `payos.debug.js=true` or `PAYOS_DEBUG_JS=true` before the JVM started.
- An API route that executes backend JavaScript has been called.
- The copied runtime JAR is fresh and contains the `payos.debug.js` support.

To verify the exact Java command currently running:

```powershell
Get-CimInstance Win32_Process -Filter "name = 'java.exe' OR name = 'javaw.exe'" |
  Select-Object ProcessId, CommandLine |
  Format-List
```

To check whether the launched JAR contains the debug support:

```powershell
javap `
  -classpath "C:\Projets\DTS\MedTech@Work\Innovation\S2M\payos-bundles\payos-runtime-1.3.0-RELEASE.jar" `
  -private `
  -c ma.s2m.payos.scripting.graalvm.PolyglotScriptingEngine |
  Select-String -Pattern "payos.debug.js|PAYOS_DEBUG_JS|inspect|isJsDebugEnabled"
```

If this command prints no matching output, the launched JAR is stale. Rebuild `payos-runtime`, copy the new JAR into the app-bundles folder, and restart PayOS.

### Handler Runs But The Debugger Does Not Attach

If the API returns normally instead of waiting, `payos.debug.js.suspend=true` was probably not active in the JVM that handled the request. Stop PayOS and restart it with the debug flags.

If `/json/list` returns `"type": "node"` but VS Code does not attach, verify that `app1/.vscode/launch.json` uses `"type": "node"`, not `"type": "chrome"`.

### Breakpoints Stay Unbound

Verify that the JavaScript file opened in VS Code is the same file path that PayOS loads from the app bundle. For example, if `bootstrap.json` points `app1` to:

```text
C:\Projets\DTS\MedTech@Work\Innovation\S2M\payos-bundles\app1
```

then set breakpoints in files under that exact folder.

### GraalVM Inspector Is Unavailable

If PayOS logs that the GraalVM inspector is unavailable, use a shaded runtime JAR that includes `org.graalvm.tools:chromeinspector-tool`, then restart PayOS.
