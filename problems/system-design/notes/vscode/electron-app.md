## What this file *is*, goal-first

**Terminal goal:** stand up exactly one Electron main-process instance and hand control to it. Everything else is a means to that end. Reading `CodeApplication` as a decision funnel:

1. **Harden the process boundary** (session security policy) — must happen before anything touches untrusted content.
2. **Stand up cross-process transport** (Electron IPC server, Node IPC server, shared process, MCP gateway) — the substrate every later service depends on.
3. **Compose the dependency-injection (DI) graph** (`initServices`) — this is the *composition root*: the one place object graphs get wired instead of self-constructing.
4. **Expose that graph over IPC** (`initChannels`) — turn in-process services into cross-process contracts.
5. **Resolve intent** (protocol URLs, CLI args, macOS `open-file` events) into a concrete "what window(s) do we open."
6. **Open windows, then degrade priority** through lifecycle phases (`Ready → AfterWindowOpen → Eventually`), deferring non-critical telemetry work until the event loop is idle.

$$
\text{startup} \;=\; \text{harden} \;\triangleright\; \text{wire}(D) \;\triangleright\; \text{resolve\_intent} \;\triangleright\; \text{open} \;\triangleright\; \text{degrade\_priority}
$$

where $\triangleright$ is sequential composition and $D$ is the service graph below.

```mermaid
flowchart TD
    A[startup] --> B[resolve machineId/sqmId/devDeviceId]
    B --> C[setupSharedProcess]
    C --> D[initServices: build DI ServiceCollection]
    D --> E[register ErrorTelemetry]
    E --> F[instantiate Agent Host starter + manager]
    F --> G[wire metered-connection telemetry]
    G --> H[resolve IProxyAuthService]
    H --> I[register UserDataProfilesHandler]
    I --> J[initChannels: expose services over IPC]
    J --> K[setupProtocolUrlHandlers]
    K --> L[setupManagedRemoteResourceUrlHandler]
    L --> M[phase = Ready]
    M --> N[openFirstWindow]
    N --> O[phase = AfterWindowOpen]
    O --> P[afterWindowOpen: crash reporter, shell env, GPU/power telemetry]
    P --> Q[schedule eventuallyPhaseScheduler, 2.5s idle]
    Q --> R[phase = Eventually]
    R --> S[eventuallyAfterWindowOpen: device-id validation, OS proxy telemetry]
```

## Formalizing the DI container (set-theoretic, then categorical)

Let $S$ be the finite set of service tokens registered in `initServices` (e.g. `IWindowsMainService`, `ITelemetryService`, …). Registration is a **partial function**

$$
\mathrm{reg} : S \rightharpoonup D
$$

into a set of descriptors $D$, where each descriptor is either an already-built instance, or a `SyncDescriptor(ctor, args)` — a *recipe*, not a value.

Because `SyncDescriptor` defers construction until some `IInstantiationService` runs it, $\mathrm{reg}(s)$ isn't an implementation of $s$ directly — it's a computation that *produces* one. That's exactly the shape of a Kleisli morphism for the thunk monad $M X = (\text{InstantiationContext}) \to X$:

$$
\mathrm{reg}(s) : 1 \longrightarrow M(\mathrm{Impl}(s))
$$

`mainInstantiationService.createChild(services)` at the end of `initServices` is then a concrete set operation: if $S_0$ is the parent's already-registered token set and $S_1$ is the set newly bound in this file, the child's registration is the **coproduct of two partial functions with disjoint domains**:

$$
\mathrm{reg}_{\text{child}} = \mathrm{reg}_{\text{parent}} \cup \mathrm{reg}_{\text{new}}, \qquad \mathrm{dom}(\mathrm{reg}_{\text{parent}}) \cap \mathrm{dom}(\mathrm{reg}_{\text{new}}) = \varnothing
$$

The child can *see* everything the parent bound (union), but the parent can never see the child's bindings — the inclusion $S_0 \hookrightarrow S_0 \cup S_1$ is monic in one direction only. That asymmetry is *why* `appInstantiationService` (child) exists at all instead of registering everything on `mainInstantiationService` directly: it scopes app-lifetime services away from earlier-phase (pre-app) services.

```mermaid
flowchart LR
    subgraph Windowing
        IWindowsMainService
        IAuxiliaryWindowsMainService
        IDialogMainService
    end
    subgraph PlatformServices["Platform services"]
        IUpdateService
        IKeyboardLayoutMainService
        IGlobalKeybindingsMainService
        INativeHostMainService
    end
    subgraph StorageWorkspace["Storage & workspace"]
        IStorageMainService
        IApplicationStorageMainService
        IBackupMainService
        IWorkspacesManagementMainService
        IWorkspacesService
        IWorkspacesHistoryMainService
    end
    subgraph Terminal
        ILocalPtyService
        IExternalTerminalMainService
        ISandboxHelperMainService
    end
    subgraph NetworkExt["Networking & extensions"]
        IURLService
        IProxyAuthService
        IMeteredConnectionService
        IExtensionHostStarter
        IExtensionsProfileScannerService
        IExtensionsScannerService
    end
    subgraph Observability
        ITelemetryService
        IDiagnosticsMainService
        IDiagnosticsService
    end
    subgraph MCPWeb["MCP & web content"]
        INativeMcpDiscoveryHelperService
        IMcpGatewayService
        IWebContentExtractorService
        IWebviewManagerService
    end
    Root["appInstantiationService = mainInstantiationService.createChild(S₁)"] --> Windowing
    Root --> PlatformServices
    Root --> StorageWorkspace
    Root --> Terminal
    Root --> NetworkExt
    Root --> Observability
    Root --> MCPWeb
```

## Security predicate: `webRequest.onBeforeRequest`

This is a **classification function** $\text{allow} : \text{Request} \to \{0,1\}$ built as a conjunction of scheme-specific guards, each itself a decision procedure over the frame ancestry chain.

```mermaid
flowchart TD
    Req[incoming webRequest] --> Scheme{scheme?}
    Scheme -->|vscode-webview| WV{isAllowedWebviewRequest: is this index.html from a main editor window?}
    WV -->|no| Cancel1[cancel]
    WV -->|yes| Next1[continue]
    Scheme -->|vscode-file-resource| VF{isAllowedVsCodeFileRequest: frame belongs to a main window?}
    VF -->|no| Cancel2[cancel]
    VF -->|yes| Next2[continue]
    Scheme -->|vscode-managed-remote-resource| MR{frame chain never passes through a webview?}
    MR -->|fails / destroyed| Cancel3[cancel]
    MR -->|passes| Next3[continue]
    Scheme -->|other| SVG{path ends in .svg AND scheme not in supportedSvgSchemes?}
    SVG -->|yes| Safe{request is xhr, or frame ancestry includes a webview?}
    Safe -->|no| Cancel4[cancel]
    Safe -->|yes| Allow[allow]
    SVG -->|no| Allow
```

## Protocol-URL resolution funnel: `handleProtocolUrl`

```mermaid
flowchart TD
    U[handleProtocolUrl uri] --> WS{path == workspace?}
    WS -->|yes| RW[rewrite to file-scheme uri from query]
    WS -->|no| PQ
    RW --> PQ[parse query params]
    PQ --> Blank{windowId == _blank?}
    Blank -->|yes| NW[shouldOpenInNewWindow = true]
    Blank -->|no| MacZero{macOS AND zero windows open?}
    MacZero -->|yes| NW
    MacZero -->|no| Ope
    NW --> Ope{resolves to file/folder/workspace openable?}
    Ope -->|yes| Blk{shouldBlockOpenable: user confirms external open?}
    Blk -->|blocked| Done1[treated as handled — no window]
    Blk -->|allowed| OW[windowsMainService.open with urisToOpen]
    Ope -->|no| NF{shouldOpenInNewWindow?}
    NF -->|yes| EW[open empty window] --> FWD[urlService.open uri inside it]
    NF -->|no| NH[return false — not handled here]
```

If you want, I can also formalize `shouldBlockOpenable`'s trust boundary (it's a nice small example of a **Galois-connection-shaped** confirm/deny gate between `Schemas` and user consent) or diagram the shared-process handshake (`connect → whenReady → MessagePortClient`) as a two-stage promise pipeline.