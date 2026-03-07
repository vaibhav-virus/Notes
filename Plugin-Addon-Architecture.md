# 🔌 Designing an Add-On / Plugin System for a C++ CAD Application
### (Autodesk Fusion 360 Style — Senior SDE Interview Reference)

---

## Table of Contents
1. [Core Philosophy](#1-core-philosophy)
2. [Architecture Overview](#2-architecture-overview)
3. [Module Breakdown](#3-module-breakdown)
4. [Plugin Lifecycle](#4-plugin-lifecycle)
5. [API Surface Design (C++ SDK)](#5-api-surface-design-c-sdk)
6. [ABI Stability & Versioning](#6-abi-stability--versioning)
7. [Shipping the SDK / DevKit](#7-shipping-the-sdk--devkit)
8. [UI Extensibility](#8-ui-extensibility)
9. [Interoperability (Interop)](#9-interoperability-interop)
10. [Security & Sandboxing](#10-security--sandboxing)
11. [Web-Based Plugin System](#11-web-based-plugin-system)
12. [Marketplace & Distribution](#12-marketplace--distribution)
13. [Interview Talking Points Summary](#13-interview-talking-points-summary)

---

## 1. Core Philosophy

> **"Expose, don't expose internals."**  
> A plugin system is a *contract* between the host application and third-party developers. The host controls stability; developers control creativity.

### Key Design Principles

| Principle | Meaning |
|---|---|
| **Stability over features** | Public API must not break between minor versions |
| **Least privilege** | Plugin should only access what it needs |
| **Isolation** | Plugin crash should not crash the host |
| **Discoverability** | SDK must be easy to explore (docs, types, samples) |
| **Composability** | Plugins can build on each other (services architecture) |

---

## 2. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      HOST APPLICATION (C++)                     │
│                                                                 │
│  ┌──────────────┐   ┌──────────────┐   ┌───────────────────┐  │
│  │  Core Engine │   │  UI Shell    │   │  Service Registry │  │
│  │  (Geometry,  │   │  (Qt/Win32/  │   │  (DI Container)   │  │
│  │   Modeling)  │   │   WebView)   │   │                   │  │
│  └──────┬───────┘   └──────┬───────┘   └────────┬──────────┘  │
│         │                  │                     │              │
│  ═══════╪══════════════════╪═════════════════════╪═══════════  │
│                    PLUGIN API LAYER                             │
│          (Stable ABI, Pure C Interface or Opaque Handles)      │
│  ═══════╪══════════════════╪═════════════════════╪═══════════  │
│         │                  │                     │              │
│  ┌──────▼──────────────────▼─────────────────────▼──────────┐  │
│  │                   Plugin Host / Loader                    │  │
│  │   (DLL/SO loading, lifecycle management, sandboxing)      │  │
│  └───────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬─────────────────────────────────┘
                                │  (Dynamic Load)
             ┌──────────────────┼────────────────────┐
             │                  │                    │
     ┌───────▼──────┐  ┌────────▼─────┐  ┌──────────▼──────┐
     │  Add-on A    │  │  Add-on B    │  │  Add-on C       │
     │  (C++ DLL)   │  │  (Python)    │  │  (Web/JS Panel) │
     └──────────────┘  └──────────────┘  └─────────────────┘
```

**Three layers:**
1. **Core Host** — The real application internals (never exposed directly)
2. **Plugin API Layer** — Stable public contract (the SDK boundary)
3. **Plugin Runtime** — Loaded modules (C++ DLLs, Python scripts, Web panels)

---

## 3. Module Breakdown

### 3.1 Host-Side Modules

```
host/
├── core/
│   ├── ModelingEngine        # B-Rep / mesh kernel
│   ├── SceneGraph            # Node hierarchy (Component, Body, Face...)
│   ├── UndoRedo              # Command pattern stack
│   └── EventBus              # Pub/sub internal to host
│
├── plugin-host/
│   ├── PluginLoader          # LoadLibrary / dlopen abstraction
│   ├── PluginRegistry        # Map<PluginId, PluginInstance>
│   ├── LifecycleManager      # Initialize / Activate / Deactivate / Unload
│   ├── ServiceRegistry       # Maps IService* to implementation
│   └── PermissionGate        # Checks plugin capability tokens
│
├── api/  (this becomes the SDK)
│   ├── include/
│   │   ├── adsk/
│   │   │   ├── core/         # Application, Document, Events
│   │   │   ├── fusion/       # Component, BRepBody, Sketch...
│   │   │   ├── cam/          # Toolpath, Operation
│   │   │   └── ui/           # CommandDefinition, Palette, Toolbar
│   └── lib/
│       └── adsk_sdk.lib / .a
│
└── ui-shell/
    ├── CommandRouter         # Maps toolbar/shortcut to handler
    ├── PanelManager          # Manages dockable panels
    └── WebViewBridge         # Hosts Chromium/WebKit for web panels
```

### 3.2 Plugin-Side Modules (What developers write)

```
my-addon/
├── src/
│   ├── Entry.cpp             # DLL entry: adsk_plugin_init()
│   ├── MyCommand.cpp         # Implements ICommand interface
│   ├── MyEventHandler.cpp    # Subscribes to host events
│   └── MyPalette.cpp         # Registers a web panel (HTML/JS)
├── resources/
│   ├── ui/                   # HTML, CSS, JS for panels
│   └── icons/                # Toolbar icons
├── manifest.json             # Plugin metadata, permissions, version
└── CMakeLists.txt            # Links against adsk_sdk.lib
```

### `manifest.json` example

```json
{
  "id": "com.mycompany.myaddon",
  "name": "My Awesome Add-on",
  "version": "2.1.0",
  "minHostVersion": "2.0.0",
  "entryPoint": "MyAddOn.dll",
  "permissions": [
    "document.read",
    "document.write",
    "ui.toolbar",
    "ui.panel"
  ],
  "apiVersion": "3"
}
```

---

## 4. Plugin Lifecycle

```
  [Discovery]
      │   Host scans plugin dirs, reads manifest.json
      ▼
  [Validation]
      │   Checks minHostVersion, permissions, signature
      ▼
  [Load]
      │   LoadLibrary("MyAddOn.dll")
      │   Resolves adsk_plugin_init symbol
      ▼
  [Initialize]
      │   adsk_plugin_init(IPluginContext* ctx) called
      │   Plugin registers commands, event handlers, panels
      ▼
  [Active / Running]
      │   User interacts; events dispatched to plugin
      │   Plugin calls host API (e.g., createBox, getSelection)
      ▼
  [Deactivate]  (plugin toggle / document close)
      │   adsk_plugin_deactivate() called
      │   Plugin should release UI registrations
      ▼
  [Unload]
      │   adsk_plugin_shutdown() called
      │   FreeLibrary()
      ▼
  [Unloaded]
```

### Entry Point Contract (C API — ABI safe)

```cpp
// In plugin DLL — must export these symbols
extern "C" {
    // Called once when loaded
    ADSK_EXPORT AdskResult adsk_plugin_init(IPluginContext* ctx);

    // Called when host wants to cleanly shutdown
    ADSK_EXPORT void       adsk_plugin_shutdown();
    
    // Optional: called when document opens/closes
    ADSK_EXPORT void       adsk_plugin_on_document_event(AdskDocEventType type);
}
```

> **Why `extern "C"` with opaque `IPluginContext*`?**  
> C++ has no stable ABI across compilers (MSVC vs Clang vs GCC) or even different versions of the same compiler. A **pure C interface** or **opaque handle pattern** guarantees that no matter what compiler the plugin developer uses, the ABI contract holds.

---

## 5. API Surface Design (C++ SDK)

The SDK wraps raw C handles in a nice C++ layer (RAII, smart pointers, iterators) — but the *binary boundary* stays C.

### Pattern: Opaque Handle + Wrapper Class

```cpp
// --- Internal host code (never shipped) ---
class BRepBodyImpl {
    // Complex internal structure
    std::vector<FaceImpl*> faces_;
    // ... lots of internal stuff
};

// --- ABI boundary (C handle) ---
typedef void* AdskBRepBodyHandle;

// C API (stable binary contract)
extern "C" {
    ADSK_API int         adsk_brep_body_face_count(AdskBRepBodyHandle h);
    ADSK_API AdskFaceHandle adsk_brep_body_get_face(AdskBRepBodyHandle h, int idx);
    ADSK_API void        adsk_brep_body_add_ref(AdskBRepBodyHandle h);
    ADSK_API void        adsk_brep_body_release(AdskBRepBodyHandle h);
}

// --- SDK C++ Wrapper (shipped in headers, wraps C API) ---
class BRepBody {
public:
    explicit BRepBody(AdskBRepBodyHandle h) : h_(h) {
        adsk_brep_body_add_ref(h_);
    }
    ~BRepBody() { adsk_brep_body_release(h_); }

    // No copy (or implement with addref)
    BRepBody(const BRepBody&) = delete;
    BRepBody& operator=(const BRepBody&) = delete;
    
    // Move OK
    BRepBody(BRepBody&& o) noexcept : h_(o.h_) { o.h_ = nullptr; }

    int faceCount() const { return adsk_brep_body_face_count(h_); }
    BRepFace getFace(int idx) const { return BRepFace(adsk_brep_body_get_face(h_, idx)); }

    // Range-based for support
    FaceIterator begin() const;
    FaceIterator end() const;

private:
    AdskBRepBodyHandle h_;
};
```

### Event System

```cpp
// Plugin subscribes to events
class MySelectionHandler : public ISelectionEventHandler {
public:
    void onSelectionChange(const SelectionEvent& e) override {
        auto selected = e.selection();
        for (auto& entity : selected) {
            if (entity.type() == EntityType::BRepFace) {
                auto face = entity.as<BRepFace>();
                // process...
            }
        }
    }
};

// In plugin init:
void adsk_plugin_init(IPluginContext* ctx) {
    auto& app = ctx->application();
    app.selectionChanged().connect(std::make_shared<MySelectionHandler>());
}
```

### Command Pattern

```cpp
// Every operation is a Command for undo/redo support
class CreateBoxCommand : public ICommand {
public:
    // Called when user clicks OK in dialog
    AdskResult execute(const CommandInputs& inputs) override {
        double width  = inputs.get<double>("width");
        double height = inputs.get<double>("height");
        double depth  = inputs.get<double>("depth");
        
        auto& design = ctx_->application().activeDocument().design();
        auto root = design.rootComponent();
        auto bodies = root.bRepBodies();
        bodies.addBox(width, height, depth);           // creates body
        return AdskResult::OK;
    }
    
    // Called on undo — host handles undo stack; command provides rollback
    AdskResult undo() override { /* ... */ }
};
```

---

## 6. ABI Stability & Versioning

### Problem
C++ vtables, name mangling, struct padding — all differ per compiler. You can't ship a `.h` with a class and expect it to work with all compilers.

### Solutions

| Strategy | When to Use |
|---|---|
| **Pure C API** (C handles + free functions) | Maximum compatibility |
| **COM-style vtable** (fixed layout virtual tables) | Windows; widely used by Autodesk |
| **Protobuf / FlatBuffers over IPC** | Cross-process plugins (strongest isolation) |
| **Header-only wrappers** | C++ convenience on top of C API |

### Versioning Strategy

```
API Version (major.minor)
  Major bump  → breaking change (rare, handled by adapters)
  Minor bump  → additive only

Deprecation Policy:
  - Mark with ADSK_DEPRECATED(version) macro
  - Keep 2 major versions; emit warnings
  - Remove in N+2

Host version check at load time:
  if (pluginManifest.minHostVersion > host.version()) {
      reject("Plugin requires newer host");
  }
  if (pluginManifest.apiVersion > host.supportedApiVersion()) {
      reject("Plugin API version not supported");
  }
```

---

## 7. Shipping the SDK / DevKit

### What to Ship

```
autodesk-fusion-sdk/
│
├── include/                          # C++ headers
│   └── adsk/
│       ├── core/Application.h
│       ├── fusion/BRepBody.h
│       ├── cam/Toolpath.h
│       └── ui/CommandDefinition.h
│
├── lib/
│   ├── win64/adsk_sdk.lib            # Import library (MSVC)
│   ├── win64/adsk_sdk.dll            # Redistributable
│   ├── linux64/libadsk_sdk.so
│   └── mac/libadsk_sdk.dylib
│
├── cmake/
│   └── FindAdskSDK.cmake             # CMake find_package support
│
├── samples/
│   ├── hello-addon/                  # Minimal working addon
│   ├── brep-analysis/                # Geometry traversal sample
│   └── custom-ui-panel/              # Web panel integration sample
│
├── docs/
│   ├── api-reference/                # Doxygen-generated or hand-written
│   ├── getting-started.md
│   ├── migration/v2-to-v3.md         # Breaking change guide
│   └── tutorials/
│
├── tools/
│   ├── addon-validator.exe           # Validates manifest.json
│   └── package-addon.exe             # Zips with signing
│
└── CHANGELOG.md
```

### CMake Integration (Plugin Developer Side)

```cmake
# Plugin developer's CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(MyAddOn)

find_package(AdskSDK 3.0 REQUIRED)

add_library(MyAddOn SHARED
    src/Entry.cpp
    src/MyCommand.cpp
)

target_link_libraries(MyAddOn PRIVATE Adsk::SDK)
target_compile_features(MyAddOn PRIVATE cxx_std_17)

# Auto-copy DLL to plugin dir for debug
add_custom_command(TARGET MyAddOn POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy
        $<TARGET_FILE:MyAddOn>
        "$ENV{ADSK_PLUGIN_DIR}/MyAddOn.dll"
)
```

### Delivery Options

| Method | Description |
|---|---|
| **Installer / Bundle** | SDK shipped with the app (Autodesk does this) |
| **NuGet / vcpkg / Conan** | Package manager for C++ libraries |
| **GitHub Releases** | Versioned zip archives with binaries + headers |
| **Python Bindings** | `pip install autodesk-fusion-sdk` via pybind11 |
| **Web SDK (npm)** | `npm install @autodesk/fusion-web-sdk` for JS |

---

## 8. UI Extensibility

### 8.1 Extension Points

```
Application UI
├── Menu Bar
│   └── Plugin can add menus / menu items
├── Toolbar / Ribbon
│   └── Plugin registers CommandDefinition → toolbar button appears
├── Context Menu (Right-click)
│   └── Plugin hooks into context menu provider; adds items conditionally
├── Properties Panel
│   └── Plugin provides custom property page for selected entity
├── Dialog / Modal
│   └── Plugin creates InputDialog with standard controls (sliders, dropdowns)
├── Palette (Dockable Panel)
│   └── Plugin hosts a WebView (HTML/CSS/JS) in a dockable panel
└── Viewport Overlay
    └── Plugin renders OpenGL/DX overlays on top of viewport
```

### 8.2 Toolbar Button Registration

```cpp
void adsk_plugin_init(IPluginContext* ctx) {
    auto& ui = ctx->application().userInterface();

    // Create command definition
    auto cmdDef = ui.commandDefinitions().addButtonDefinition(
        "MyAddOn_CreateBoxCmd",          // unique ID
        "Create Smart Box",              // display name
        "Creates a parameterized box",   // tooltip
        "./resources/icons"              // icon folder (16, 32, 64 px PNGs)
    );

    // Hook execute event
    cmdDef.commandCreated().connect([](CommandCreatedEvent& e) {
        auto& cmd = e.command();
        cmd.execute().connect([](CommandEvent& ex) {
            // run logic
        });
    });

    // Add to toolbar
    auto panel = ui.allToolbarPanels().itemById("SolidCreatePanel");
    panel.controls().addCommand(cmdDef);

    // Remember for cleanup
    registeredControls_.push_back(panel.controls().itemById("MyAddOn_CreateBoxCmd"));
}
```

### 8.3 Web Panel (Palette) — HTML UI in the app

```cpp
// C++ side: register the palette
auto palette = ui.palettes().add(
    "MyAddOn_Palette",
    "My Add-On Panel",
    "resources/ui/index.html",       // HTML file bundled in addon
    true,   /* is visible */
    true,   /* show close button */
    true,   /* is resizable */
    800, 400
);

// Listen for messages from the HTML/JS side
palette.incomingFromHTML().connect([](HTMLEvent& e) {
    std::string action = e.data();   // JSON string from JS
    // parse and handle
    if (action == "GET_BODIES") {
        auto& bodies = ctx_->application()
                           .activeDocument()
                           .design()
                           .rootComponent()
                           .bRepBodies();
        std::string json = serializeBodies(bodies);
        palette.sendInfoToHTML("bodies_list", json);  // push to JS
    }
});
```

```javascript
// resources/ui/index.js (runs inside the WebView)
window.adsk = window.adsk || {};

// Called when C++ pushes data
adsk.fusionSendData = function(action, data) {
    if (action === 'bodies_list') {
        renderBodyList(JSON.parse(data));
    }
};

// Call C++ from JS
function requestBodies() {
    adsk.fusionSendData = window.adsk.fusionSendData; // keep reference
    window.webkit.messageHandlers.adsk.postMessage('GET_BODIES');
    // or on Windows: window.chrome.webview.postMessage('GET_BODIES')
}
```

### 8.4 Viewport Overlay (OpenGL / DX hooks)

```cpp
class MyOverlayRenderer : public ICustomGraphicsRenderer {
public:
    void render(ICustomGraphicsRenderContext& ctx) override {
        // Access the 3D viewport and draw custom geometry
        auto& graphics = ctx.customGraphicsGroups();
        auto group = graphics.add();
        
        // Draw measurement annotations, bounding boxes, etc.
        auto line = group.addLines(startPoint, endPoint, false);
        line.color(CustomGraphicsSolidColorEffect::create(Color::red()));
    }
};

// Register
app.activeViewport().customGraphics().add(
    std::make_shared<MyOverlayRenderer>()
);
```

---

## 9. Interoperability (Interop)

### 9.1 C++ ↔ Python Interop

**Why Python?**  
Many CAD users are engineers who prefer scripting. Autodesk Fusion exposes full Python API.

**Approach: pybind11**

```cpp
// In SDK build system, expose C++ SDK to Python
#include <pybind11/pybind11.h>
namespace py = pybind11;

PYBIND11_MODULE(adsk_core, m) {
    py::class_<Application>(m, "Application")
        .def("activeDocument", &Application::activeDocument)
        .def("version",        &Application::version);

    py::class_<BRepBody>(m, "BRepBody")
        .def("faceCount",  &BRepBody::faceCount)
        .def("getFace",    &BRepBody::getFace);
    // ...
}
```

```python
# Plugin developer writes Python
import adsk.core, adsk.fusion

def run(context):
    app  = adsk.core.Application.get()
    ui   = app.userInterface()
    doc  = app.activeDocument()
    design = doc.design()
    root = design.rootComponent()

    box = root.bRepBodies().addBox(adsk.core.Point3D.create(0,0,0), 10, 10, 10)
    ui.messageBox(f"Created body: {box.name}")
```

### 9.2 C++ ↔ Web / JavaScript Interop

```
 C++ Host App
      │
      │  WebView2 (Windows) / WKWebView (macOS) / CEF (cross-platform)
      │
 ┌────▼────────────────────────────┐
 │         WebView (Chromium)       │
 │                                  │
 │  HTML/CSS/JS Plugin UI           │
 │  ← postMessage →                 │
 └──────────────────────────────────┘

Message passing is JSON over WebView bridge.
Optionally: WebSocket to a local HTTP server run by C++ host.
```

**Option A: Embedded WebView (in-process, tight UI integration)**
- WebView2 (Windows), WKWebView (macOS), CEF (cross-platform)
- JS calls `window.chrome.webview.postMessage(jsonString)`
- C++ receives in `WebMessageReceived` event handler

**Option B: Local Loopback (out-of-process, REST or WebSocket)**
```
Plugin C++ DLL ──starts──► Local HTTP server (localhost:PORT)
Plugin Web UI (browser)  ──REST/WS──► localhost:PORT
Host app exposes REST API  ◄─── Plugin calls /api/getbodies etc.
```

### 9.3 Cross-Language Type Contracts

```
Define types in a neutral format:
  - JSON Schema  → validated messages between C++ and JS
  - Protobuf     → for performance-critical IPC
  - OpenAPI spec → if using REST bridge

Example JSON schema for a Body:
{
  "type": "object",
  "properties": {
    "id":       { "type": "string" },
    "name":     { "type": "string" },
    "faceCount": { "type": "integer" },
    "volume":   { "type": "number" }
  }
}
```

### 9.4 Out-of-Process Plugins (Strong Isolation)

```
[Host Process]                    [Plugin Process]
     │                                   │
     │  ← named pipe / gRPC / COM RPC →  │
     │                                   │
IPluginProxy ──────────────────► IPlugin (actual impl)
(marshals calls)                (isolated; crash safe)
```

- If plugin crashes → host detects via heartbeat; marks plugin faulted
- Performance cost: serialization on every call
- Used for untrusted / marketplace plugins

---

## 10. Security & Sandboxing

### Permission Model

```
Capabilities declared in manifest.json:
  "permissions": [
    "document.read",       // read geometry, metadata
    "document.write",      // modify geometry
    "ui.toolbar",          // add toolbar buttons
    "ui.panel",            // open web panels
    "network.outbound",    // make HTTP calls (requires user consent)
    "filesystem.read",     // read local files (sandboxed dir only)
    "execute.subprocess"   // very restricted; user must explicitly grant
  ]
```

### Code Signing

```
Plugin must be signed with developer certificate.
Host verifies signature before loading.

Pipeline:
Developer builds DLL →
  package-addon.exe signs the package (code signing cert) →
  Submit to Autodesk App Store →
  Autodesk re-signs after review →
  Published to marketplace
```

### Sandboxing Levels

| Level | Description | Use case |
|---|---|---|
| **In-process (trusted)** | Same process as host; full speed | Autodesk first-party |
| **In-process (permissioned)** | Same process; API calls gated by permission check | Verified marketplace plugins |
| **Out-of-process** | Separate process; IPC bridge | High-risk / untrusted plugins |
| **Containerized** | Docker/Sandbox API; filesystem/network isolated | Cloud execution of scripts |

---

## 11. Web-Based Plugin System

For the **web version** of the CAD app (e.g., Fusion 360 browser app):

### Architecture

```
            Browser Tab (Main App)
            ┌──────────────────────────────────────┐
            │  React / Angular Host Shell           │
            │  ┌────────────────────────────────┐  │
            │  │  Canvas / WebGL Viewport        │  │
            │  └────────────────────────────────┘  │
            │  ┌───────────────┐  ┌─────────────┐  │
            │  │  Host UI      │  │  Plugin      │  │
            │  │  Components   │  │  Iframe/     │  │
            │  │               │  │  Web Worker  │  │
            │  └───────────────┘  └──────┬───────┘  │
            └─────────────────────────── │ ─────────┘
                                         │  postMessage
            ┌────────────────────────────▼──────────┐
            │        Plugin Sandbox (iframe)         │
            │   Plugin HTML + JS (3rd party code)   │
            └───────────────────────────────────────┘
```

### Web SDK Design

```javascript
// @autodesk/fusion-web-sdk (npm package)
// Plugin developer's index.js:

import { Application, SelectionEvents } from '@autodesk/fusion-web-sdk';

const app = Application.getInstance();

app.onReady(async () => {
    const doc   = await app.activeDocument();
    const design = await doc.design();
    const root  = await design.rootComponent();
    const bodies = await root.bRepBodies();
    
    console.log(`Body count: ${bodies.length}`);
    
    // Subscribe to events
    app.events.on(SelectionEvents.SelectionChanged, (e) => {
        const entities = e.selection;
        renderSidebar(entities);
    });
});
```

### Key Web Plugin Patterns

```
Pattern 1: Iframe Isolation
  - Plugin runs in <iframe sandbox="allow-scripts">
  - Communicates via window.postMessage with typed protocol
  - Host validates messages; rejects unauthorized operations

Pattern 2: Web Worker (compute)
  - Offload heavy geometry computation to Web Worker
  - Worker receives geometry data as ArrayBuffer (SharedArrayBuffer for perf)
  - Result posted back to main thread

Pattern 3: Service Worker (offline/caching)
  - Plugin can register service worker for offline asset caching

Pattern 4: WebAssembly (C++ logic in browser)
  - Port plugin logic to WASM via Emscripten
  - Share geometry algorithms between desktop C++ and web WASM
```

### WASM Compilation of C++ Plugin for Web

```cmake
# Emscripten build
emcmake cmake .. -DTARGET_WASM=ON
emmake make

# Outputs: my_plugin.wasm + my_plugin.js (JS glue)
```

```javascript
// Load WASM module in browser plugin
import createModule from './my_plugin.js';

const module = await createModule();
const result = module.computeIntersection(body1Data, body2Data);
```

### API Consistency: Desktop ↔ Web

```
Goal: Same plugin (or minimal changes) runs on desktop and web.

Approach:
  Desktop C++  →  calls ADSK C++ SDK
  Web plugin   →  calls ADSK JS SDK (same conceptual API, different impl)

Both SDKs model the same object hierarchy:
  Application → Document → Design → Component → BRepBody → BRepFace → ...

Web SDK wraps REST API calls to cloud backend:
  Body.faceCount() → fetch('/api/v3/bodies/{id}/faces/count')
```

---

## 12. Marketplace & Distribution

### Flow

```
Developer Side:
  1. Build plugin (C++ DLL / Python scripts / Web bundle)
  2. Run addon-validator.exe (checks manifest, symbols, permissions)
  3. Sign with code cert
  4. Submit to Autodesk App Store (ZIP + metadata + screenshots)
  5. Autodesk review (security scan, API usage audit)
  6. Publish

User Side:
  1. Browse App Store (web or in-app)
  2. Click Install → downloaded to %APPDATA%\Autodesk\Fusion 360\Addins\
  3. Host discovers and loads on next launch (or hot-reload if supported)
  4. Uninstall = move to quarantine, then delete
```

### Plugin Discovery

```cpp
// Host scans these directories in order:
std::vector<fs::path> pluginSearchPaths = {
    getInstallDir() / "plugins" / "builtin",     // First-party
    getAppDataDir() / "Addins",                   // User-installed
    getSystemDir()  / "Autodesk" / "Addins",      // Enterprise-wide
};

for (auto& dir : pluginSearchPaths) {
    for (auto& entry : fs::directory_iterator(dir)) {
        if (auto manifest = loadManifest(entry / "manifest.json")) {
            pluginLoader_.queue(*manifest, entry);
        }
    }
}
```

---

## 13. Interview Talking Points Summary

### "How would you design a plugin system for a C++ CAD app?"

> "I'd define a **stable ABI boundary** using a pure C interface (or opaque handle pattern), since C++ has no stable ABI. Above that, I'd ship a header-only C++ wrapper for convenience. Plugins are `.dll`/`.so` files discovered via manifest files, loaded with `LoadLibrary`/`dlopen`, and their lifecycle (init/shutdown) managed by a PluginHost component. I'd version the API with major/minor semantics, deprecation macros, and minimum host version checks."

### "How would you handle UI extensibility?"

> "I'd expose declarative extension points — toolbar, context menu, properties panel — where plugins register `CommandDefinition` objects. For rich UI, I'd allow plugins to host a `WebView` panel (using CEF or WebView2), communicating with the C++ host via **structured JSON messages** over the WebView bridge. This lets plugin developers use any web framework (React, Vue) while keeping host-side logic in C++."

### "How would you ensure plugin crashes don't affect the host?"

> "**Two strategies**: For trusted/verified plugins, they run in-process — fast but risky. For untrusted/marketplace plugins, I'd run them in a **separate process** with IPC (named pipes or gRPC). The host subscribes to a heartbeat; if it stops, the plugin is marked faulted and evicted. The key is that the host's document model is authoritative — plugins operate through the API layer, so any partial state from a faulted plugin can be rolled back via the undo stack."

### "What about web-based versions of the same plugins?"

> "I'd define the object model **once** (Component→BRepBody→BRepFace hierarchy) and implement it in both C++ (desktop SDK) and JavaScript (web SDK). The web SDK proxies calls to a cloud backend via REST. Plugins that contain pure-logic algorithms can be compiled to **WASM** via Emscripten and run in the browser. Web plugin isolation uses `<iframe sandbox>` + `postMessage`. Ideally, plugin authors write once and target both platforms with minimal porting."

### "How do you ship the SDK?"

> "I'd ship prebuilt binaries (`.lib`/`.so`/`.dylib`), headers, a CMake `find_package` module, Python bindings (via pybind11), and a web SDK (npm package). The package includes a validator tool, sample projects, and migration guides. API stability is enforced by CI — any PR that changes a public header triggers an ABI compatibility check (e.g., `libabigail` or `abi-compliance-checker`)."

---

*These notes are tailored for a Senior SDE interview at Autodesk Fusion 360. Focus on the design decisions and trade-offs, not just the implementation details.*
