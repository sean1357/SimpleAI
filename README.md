SimpleAI
A lightweight Windows launcher for local GGUF model servers. Runs llama.cpp and TensorSharp side by side, each with its own tab, its own port, and its own process. Includes a WebView2 chat panel that auto-navigates to whichever server is running, a Hugging Face browser for model discovery, and a live diagnostics tab that shows exactly what hardware and backends are available on the machine.

Features
Two independent servers — llama.cpp on 8080, TensorSharp on 5000. Both can run at once, or either alone.

Automatic backend detection — CPU, CUDA, ROCm, Vulkan, SYCL, OpenCL. Only the backends your hardware supports are offered.

Root folder classification — point at a llama.cpp or TensorSharp root, and the launcher finds the actual backend folder containing the server executable.

Content-based executable resolution — the launcher locates llama-server.exe / TensorSharp.Server.Host.exe by walking the folder tree, not by assuming a fixed layout.

Clean, deterministic logs — three lines on start (Launching, Model, Connected or Failed to load), two lines on stop (Stopping..., Stopped.).

WebView2 chat panel — starts as a placeholder, navigates to the running server's UI on connect, and switches to the Chat tab automatically.

Hugging Face browser — scan, filter, and download GGUF models without leaving the app.

Live diagnostics — system, GPU, backends, environment variables, current model files, and port status, refreshed every time you open the tab.

Deterministic process cleanup — killing a server logs Stopped. exactly once, whether the user clicked Stop, the model failed to load, or the server crashed on its own.

Requirements
Windows 10 or later (x64)

.NET 8 SDK (or the runtime, if you only want to run a build)

WebView2 Evergreen Runtime — required for the Chat tab. Windows 10/11 usually ships it; if not, install from Microsoft's download page.

One or both server runtimes:

llama.cpp — download a release from github.com/ggml-org/llama.cpp/releases. Pick the build that matches your GPU (CUDA, Vulkan, ROCm, SYCL) or the CPU build.

TensorSharp — download from github.com/zhongkaifu/TensorSharp/releases.

A GGUF model file — see huggingface.co/models?library=gguf or use the built-in Hugging Face tab.

Building
text
git clone <this-repo> SimpleAI
cd SimpleAI
dotnet restore
dotnet build -c Release
The build produces bin\Release\net8.0-windows\SimpleAI.exe.

Running
Double-click SimpleAI.exe, or:

text
dotnet run --project SimpleAI.csproj
You'll see six tabs: Chat, Llama, Tensor, Hugging Face, Diagnostics, About.

Using the launcher
Llama tab
Click Browse Folder... and pick your llama.cpp root — for example C:\Dev\llama.cpp. The launcher finds the subfolder containing llama-server.exe and selects the matching backend.

Click Browse GGUF... and pick a model file.

Adjust the backend, port, context size, GPU layers, and thread count if needed.

Click Start Server.

The launcher:

Writes Launching llama.cpp (<backend>) on <host>:<port> to the tab log.

Writes Model: <path>.

Waits for the server's HTTP endpoint to answer.

Writes Connected. and navigates the Chat tab to the server's web UI.

If the model fails to load:

text
Launching llama.cpp (ggml_cpu) on 0.0.0.0:8080
Model: C:\Dev\model\broken.gguf
Failed to load broken.gguf
Stopped.
Click Stop Server to terminate the process. Two more lines appear:

text
Stopping...
Stopped.
Tensor tab
Same flow, on port 5000, for TensorSharp. The launcher knows TensorSharp uses --model instead of -m, and that its WebView UI lives at /index.html. Everything else behaves identically.

Chat tab
When a server reaches Connected., its web UI loads here. If you start llama.cpp and then TensorSharp, the Chat tab follows whichever server connected most recently. Both servers keep running — starting one doesn't stop the other.

Hugging Face tab
Scan Hugging Face Hub for GGUF models, filter by model family (Llama, Qwen, Mistral, DeepSeek, Gemma, and dozens more), and download a file to a folder of your choice. Progress is shown live.

Diagnostics tab
A full snapshot of the machine:

text
System
──────────────────────────────
├─ OS               Microsoft Windows NT 10.0.19045.0
├─ .NET             8.0.0
├─ Machine          DESKTOP-ABC
├─ User             ken
├─ Process          x64
├─ OS arch          x64
├─ CPU cores        16
├─ Working dir      C:\Dev\SimpleAI
└─ System dir       C:\Windows\System32

GPU
──────────────────────────────
├─ Vendors          nvidia, intel
├─ NVIDIA GeForce RTX 4070
│  ├─ Vendor        NVIDIA
│  ├─ VRAM          12.0 GB
│  ├─ Driver        31.0.15.4601
│  └─ Status        OK
└─ Intel UHD Graphics 630
   ├─ Vendor        Intel Corporation
   ├─ VRAM          1.0 GB
   ├─ Driver        27.20.100.9316
   └─ Status        OK

Backends
──────────────────────────────
├─ Llama            ggml_cpu, ggml_cuda, ggml_vulkan
├─ Tensor           ggml_cpu, ggml_cuda, ggml_vulkan
├─ NVIDIA           True
├─ ROCm             False
├─ SYCL             False
├─ OpenCL           True
└─ Vulkan           True

Environment
──────────────────────────────
└─ Note             (no GPU/backend-affecting variables set)

Models
──────────────────────────────
├─ Llama
│  Gemma_4_12B_Q6_K.gguf
│  ├─ Size            10,485,760,000 bytes (9.77 GB)
│  ├─ Modified        2025-01-14 19:32:11
│  └─ Path            C:\Dev\model\Gemma_4_12B_Q6_K.gguf
└─ Tensor
   (not set)

Ports
──────────────────────────────
├─ 8080 (Llama)     free
└─ 5000 (Tensor)    free
Change Font lets you resize the text. The tab refreshes automatically when you switch to it.

About tab
Project info and links to llama.cpp releases, TensorSharp releases, the WebView2 runtime, the Visual C++ redistributable, and Hugging Face.

Architecture
The launcher is intentionally simple. Each tab is a UserControl that owns its own state and doesn't depend on the others.

text
MainForm
 ├─ tabChat            WebView2 host
 ├─ tabLlama           llama.cpp lifecycle
 ├─ tabTensor          TensorSharp lifecycle
 ├─ tabHuggingFace     HF scan/download
 ├─ tabDiagnostics     live environment snapshot
 └─ tabAbout           info and links
tabChat exposes NavigateAsync(url). The server tabs call this when a model connects.

tabLlama and tabTensor are structurally identical except for the executable name, the argument format, and the chat URL. Each holds a Process, a _stopping flag, a _failureReported flag, and a _exitLogged flag.

tabDiagnostics receives references to tabLlama and tabTensor in its constructor so it can read their current model paths and ports.

No cross-tab blocking. Starting llama.cpp does not prevent starting TensorSharp. Each server uses its own port.

The exit-logging pattern
Every server process is tracked by exactly one Stopped. log line, written by whichever code path first detects that the process has ended:

btnStop_Click — logs Stopped. itself after killing the process, and claims the exit flag so the Exited handler skips.

WaitForHttpAsync's !alive branch — logs Stopped. when the process dies during startup polling.

Process.Exited handler — logs Stopped. when the process dies later, during a live session.

A single _exitLogged flag makes each path idempotent. This design exists because Windows does not reliably raise Process.Exited for very short-lived processes, so the launcher cannot depend on that event for the critical Stopped. line.

Project layout
text
SimpleAI.csproj
Program.cs

MainForm.cs
MainForm.Designer.cs

tabChat.cs
tabChat.Designer.cs

tabLlama.cs
tabLlama.Designer.cs

tabTensor.cs
tabTensor.Designer.cs

tabHuggingFace.cs
tabHuggingFace.Designer.cs

tabDiagnostics.cs
tabDiagnostics.Designer.cs

tabAbout.cs
tabAbout.Designer.cs
Requires the Microsoft.Web.WebView2 NuGet package:

xml
<PackageReference Include="Microsoft.Web.WebView2" Version="1.0.2792.45" />
Notes
Model loading time is dominated by disk read speed and backend efficiency. A 9B Q8_0 on CPU loads in 3–4 seconds from an NVMe SSD. A 27B Q8_0 takes 3–4 minutes on CPU. GPU backends are much faster when they work.

Memory is released cleanly. The launcher's companion Memory Probe tool measured Windows reclaiming 9 GB of committed memory in under half a second after process exit. There is no deferred cleanup.

Backend names in the combo match folder names. If your llama.cpp root has a ggml_cuda folder containing llama-server.exe, that's the name the combo shows. If you have a custom folder like ggml_amd, browse to it and the launcher will add and select it automatically.

If ggml_amd or any GPU backend runs slower than expected, it may be falling back to CPU. Run the server manually from a command prompt with the same arguments and look for the offloading X layers to GPU line in the output.

Port conflicts are not automatically resolved. If port 8080 is already in use, change the port in the Llama tab before starting.

License
Add your own license here.

Credits
llama.cpp — github.com/ggml-org/llama.cpp

TensorSharp — github.com/zhongkaifu/TensorSharp

WebView2 — Microsoft Edge WebView2

GGUF models — Hugging Face

