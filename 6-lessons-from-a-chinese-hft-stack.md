# 6 Lessons from a Chinese HFT Stack Built from Source

The linker said the library was missing. The library was sitting right there — the `.a` file on disk, its symbols listable, everything checking out. But the build kept dying on `undefined reference to YAML::LoadFile`, and the evening went to understanding why: the linker wasn't exactly lying. It was reporting the symptom in the most misleading way possible.

That is the first lesson of QuantFabric: a complete Chinese high-frequency trading stack — market data gateway, risk gate, order gateway, strategy engine — MIT-licensed, eleven modules, building from source on an ordinary machine. Production trading systems are almost never open. This one you can clone. Six things the build teaches:

## 1. The architecture was hiding in the channel names

The first thing that strikes you isn't code — it's names. The shared-memory channels are called `MarketServer`, `OrderServer`, and `RiskServer`, and those three names *are* the architecture document:

- Ticks get published to `MarketServer`
- Strategies write orders to `OrderServer` (suffixed per account)
- Every order takes a round-trip through `RiskServer`, where the risk engine writes PASS/NOPASS back into the same channel

No service mesh, no discovery protocol, no config spaghetti to decode. Read the channel names, draw the dataflow. It works as a test for any system: if a new hire can't sketch your system from your queue and topic names alone, your names are wrong.

## 2. Microservices, minus the network

Each concern is its own OS process — market data, risk, orders, strategy — the standard microservices diagram. But there's no HTTP, no gRPC, no broker between them. On one colocated server, everything rides System V shared memory: a pair of SPSC lock-free queues per channel, with both publish-subscribe (market data fan-out) and request-reply (risk checks, order submission) semantics. Queue heads and tails sit on 64-byte-aligned cache lines to dodge false sharing.

The author reports 1–5μs IPC latency, about a tenth of loopback sockets — treat that as their number on their hardware, not a universal promise. The design instinct is what matters: when your processes share a machine, a socket is just a slow shared-memory queue with extra steps. Cross-machine traffic (GUI to server, server to watcher) still goes over TCP with a length-prefixed binary Pack protocol — shared memory where it's fast, sockets where it's necessary, one unified `PackMessage` definition for both.

## 3. Every exchange adapter is a runtime plugin

China's brokerage APIs are a fragmented mess — CTP, REM, YD for futures; OES, XTP, Tora for stocks — each with its own structs, callbacks, and quirks. Both gateways handle it the same way: `dlopen` the adapter `.so` at startup. Switching brokers means dropping a different shared object into a directory, not recompiling the system.

Each adapter does exactly one job: normalize. Convert the broker's native tick into a canonical `TFutureMarketData` or `TStockMarketData`, stamp receive timestamps, push into a lock-free queue. The callback thread translates and enqueues, nothing else; a main thread fans out to shared memory and the monitoring path. Every "multi-exchange" integration — crypto included — eventually converges on this shape. Here it's just explicit: C++, `.so` files, no abstraction theater.

## 4. The ABI tax

Back to the linker error. The root cause of that phantom missing library: the *entire codebase* compiles with `-D_GLIBCXX_USE_CXX11_ABI=0` — the old C++ ABI — because every Chinese broker API ships as a prebuilt binary compiled against the old ABI. A yaml-cpp built with the new ABI has the symbol, just name-mangled differently. The linker saw a stranger.

The project README devotes a full section to this — rebuild yaml-cpp from source with the matching flag or nothing links — and it's the most transferable lesson in the whole build: a constraint at the edge (vendor binaries you can't rebuild) propagates inward until it dictates the compiler flags of all eleven modules. Dependency vendoring gets all the attention. ABI discipline is the quieter, harder version of the same problem, and almost nobody writes about it.

## 5. Python gets the same shared memory as C++

You'd expect a thin wrapper. Instead, the shared-memory IPC framework is exported through pybind11, so a Python strategy opens the same `MarketServer` channel, reads the same ticks, and submits through the same `OrderServer` channel as the C++ engine. No sidecar, no REST shim, no serialization hop. The sample strategy is a moving-average crossover in a `BaseEngine` subclass, and the docs are honest about the trade: millisecond-level tick-to-order in Python for mid-frequency work, the C++ engine when you need microseconds.

The broader point for anyone building infra: "Python support" usually means a JSON API bolted on afterward. Here the core IPC primitive was designed language-agnostic from day one, because the binding was built alongside the queue, not after it.

## 6. The whole thing runs on one box

The companion XTest repo ships prebuilt binaries and a `run.sh` per component. On a single CentOS machine you can stand up the full topology — watcher, market gateway, risk engine, order gateway, strategy — against simulated feeds, and watch ticks flow from `MarketServer` through your strategy, out `OrderServer`, through the `RiskServer` round-trip, into the gateway. The deployment docs even cover the unglamorous parts: `isolcpus` isolation, pinning gateway and order threads to specific cores, raising `kernel.shmmax`.

Most "HFT" content is theory or a backtester in a notebook. Running the real multi-process topology — risk gate included — on one machine is what makes this a learning tool instead of just a codebase.

## The pattern underneath

Every strange choice traces back to a constraint: vendor binaries forced the ABI discipline; exchange fragmentation forced the plugin model; colo deployment forced shared-memory-everything; a small team forced the one-box deployment. Constraints didn't limit this architecture. They *are* the architecture.

Next in the series: inside the risk gate — how pre-trade risk checks run as request-reply over shared memory with hot-reloadable limits, and what that pipeline looks like when every microsecond is budgeted.

---

*QuantFabric is MIT-licensed and public on GitHub (11 modules: Utils, XAPI, XServer, XWatcher, XMarketCenter, XRiskJudge, XTrader, XDataPlayer, XQuant, SHMServer, XMonitor). The project also has a commercial closed-source variant outside the public repo, not covered here. Latency figures are the author's reported numbers on their hardware, not independently verified.*
