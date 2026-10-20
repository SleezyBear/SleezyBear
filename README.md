# Hey, I'm Madhav 👋

Here's what I've been building. Most of it lives close to the hardware: getting models to run fast, small and reliably on real machines, and proving it with measurements rather than vibes.

---

## 📱 [ondevice-medgemma-arm64](https://github.com/SleezyBear/ondevice-medgemma-arm64)

**A 4B-parameter multimodal model running fully offline on a mid-range Android phone.**

You give it a chest X-ray. MedGemma 1.5 4B looks at the image and writes a structured report (findings and impression), then IndicTrans2 adapts the report to Hindi. Everything runs on the phone's CPU with no network connection, through a Kotlin → JNI → C++ → llama.cpp/libmtmd pipeline.

The hard part was making it fit and making it fast.

**Making it fit**
- Explicit model lifecycle in C++: the model and vision projector load once, while every per-request resource (image, context, sampler, multimodal chunks) is released after each request
- The vision projector can be dropped as soon as the image has been processed, before text generation starts
- Optional dynamic context sizing, so the context is sized to the actual prompt instead of a fixed maximum
- MedGemma is fully unloaded before the translation model loads, and a device test checks the two never overlap
- Memory is tracked down to individual file mappings from `/proc/self/smaps`: model weights, projector, native libraries and anonymous memory
- Result: **2.83 GiB peak memory for the whole pipeline**, under a 4 GiB ceiling, on a Samsung Galaxy A70

**Making it fast**
- Android's baseline `arm64-v8a` build can't assume ARMv8.2 DOTPROD, so I added **SDOT dot-product kernels for `Q8_0`, `Q4_K` and `Q6_K`** in separately compiled objects. At runtime they're selected only if the CPU reports the feature and they pass a numerical parity check against the generic kernel on attention- and FFN-shaped inputs; otherwise the generic path is used.
- Routed llama.cpp's flash-attention tiles to a **4×16 NEON GEMM micro-kernel** path, leaving every other GEMM caller unchanged
- Built a host-side A/B harness over `adb` and `logcat` to measure every kernel, context and memory change on the physical phone

**Quantization study**

Standard quantization metrics look at one input at a time. This study asks a different question: when a model sees several X-ray views together, how much does quantization damage the way it *combines* them? It measures that with Möbius interaction coefficients over view coalitions.

- Built a llama-imatrix variant that feeds images through libmtmd, so the importance matrix is calibrated on real image + report pairs
- Ran a per-layer sensitivity scan and built a mixed-precision recipe (`Q4_K_S` base with selected layers at `q6_k`)
- Against stock `Q4_K_M`: **15.6% lower interaction distortion** and **20.8% lower single-view distortion**, better in 12 of 12 held-out studies (p = 0.0005), at 13% larger file size
- Negative results are reported too, including the recipes that didn't work

`C` `C++` `Kotlin` `JNI` `llama.cpp / ggml` `ARM NEON` `Android NDK` `ONNX Runtime` `Python`

---

## ⚙️ [InferenceOS](https://github.com/SleezyBear/InferenceOS)

**An autonomous optimizer for local LLM inference that works out what's slow before it tries to fix anything.**

Most "speed up your local LLM" advice is a list of flags to try. InferenceOS treats it as a measurement problem: profile the machine, watch the running model, diagnose the real bottleneck, try only the changes that could fix it, and keep a change only if it measurably helps without changing the answers.

**What it looks at**
- **Hardware:** CPU features and cache sizes through CPUID, GPUs through WMI plus real Vulkan device enumeration and CUDA/ROCm runtime checks, memory bandwidth through a multi-threaded copy probe, and live load through Windows performance counters
- **The model:** a dependency-free GGUF reader gives layer counts, KV heads and tensor sizes; a GQA-aware KV-cache estimator matches what the engine actually allocates to within 5%
- **The engine:** Vulkan per-operation GPU timers show where each generated token's time goes

**What it changes**
- How many layers go on the GPU
- Which individual tensors stay on the CPU, for example a mixture-of-experts model's expert weights or the output layer, with placement confirmed from the engine's own load log
- CPU thread count and memory mapping

**How it decides**
- Current and candidate setups are measured from fresh loads in alternating blocks (A B, B A, …), so slow drift of the machine cancels out
- Runs confounded by battery power or clock throttling are thrown out instead of being read as evidence
- Every candidate has to keep the model's answers unchanged on known questions
- Nothing is kept without your approval, and every change is recorded with its exact rollback

**How it stays safe**
- The engine runs inside a Windows Job Object, so a crash or timeout never leaves an orphaned process
- Process identity is checked by PID and creation time, so a recycled PID is never mistaken for the original server
- A crashed server is recovered at most once; a second failure stops cleanly instead of looping
- **3,000+ tests** and strict `mypy` type checking

`Python` `llama.cpp` `Vulkan` `Win32 APIs` `SQLite` `pydantic`

---

## 🗄️ [autonomous-mongodb-optimizer](https://github.com/SleezyBear/autonomous-mongodb-optimizer)

**A MongoDB tuner that proves a change helps before it touches production, and can always undo it.**

Query patterns drift, and indexes that made sense a year ago stop matching real traffic. This system watches a live MongoDB deployment, works out which indexes the current workload needs, tests them on a copy of the data, and only deploys what clears a statistical bar.

**How a run works**
1. Freeze the current query shapes and index statistics into an immutable snapshot (query literals are stripped)
2. Diagnose bottlenecks from that snapshot
3. Generate index candidates by fixed rules from each query's structure (equality, sort, range)
4. Let a local LLM rank them; it can reorder candidates but can't add, rename, remove or execute anything
5. Benchmark each candidate on an isolated physical copy of the data
6. Admit it only if paired bootstrap tests (10,000 resamples, one-sided 95% bounds) show at least a 5% gain and no protected metric gets worse
7. Get human approval, which is tied to the exact evidence and expires
8. Deploy, monitor, and roll back automatically if things go wrong

**Under the hood**
- Durable PostgreSQL orchestration: workers claim jobs with leases and heartbeats, retry with backoff, and recover after a crash from persisted state
- A PostgreSQL advisory lock per target, so only one change touches a database at a time
- A SHA-256 hash-chained ledger records every production change alongside its exact inverse, and rollback only touches indexes the ledger proves the optimizer created
- Argon2 passwords, JWT sessions with refresh rotation, and AES-256-GCM for stored secrets
- Past outcomes are kept as pgvector embeddings and only ever used to reorder candidates, never to loosen a check
- A 60-second soak test served **4,444 API requests with zero failures at 90.8 ms p95**

`Python` `FastAPI` `PostgreSQL` `pgvector` `MongoDB` `Ollama` `React` `Docker`

---

## 🔍 [verified-multidoc-qa](https://github.com/SleezyBear/verified-multidoc-qa)

**Question answering over technical documents that checks your question before answering it.**

Ask a normal RAG system "which physical cluster was used to deploy X?" about a paper that only ran a simulation, and it will usually invent a cluster or give up. This pipeline breaks the question into what it takes for granted, looks for evidence that supports or refutes each assumption, and leads its answer with any correction.

- **Planning:** the LLM splits a question into sub-questions, entities and assumptions, with one sentence a document would contain if each assumption holds and one if it doesn't
- **Retrieval:** dense bi-encoder recall for meaning plus a BM25 index for exact and rare terms that dense retrieval buries
- **Reranking:** a cross-encoder with per-facet coverage guarantees, so the best chunk for every part of the question survives
- **Verification:** each assumption is judged supported, contradicted or not addressed, with the deciding quote checked word for word against the source
- **Generation:** a two-stage reader that reasons over the evidence only, then commits to a clean answer
- **Self-healing:** if the answer says the evidence is missing, the pipeline tries one round of corrective retrieval
- **Ingestion:** table-aware PDF extraction, footnote resolution, reconstruction of flattened fractions, and caches keyed by file, code and model hash so stale results can't sneak in
- Fully local and deterministic: one LLM through Ollama, temperature 0, fixed seeds

`Python` `sentence-transformers` `BM25` `cross-encoders` `Ollama` `PyMuPDF`

---

## 📝 [faithful-multidoc-summarization](https://github.com/SleezyBear/faithful-multidoc-summarization)

**Query-focused summarization across several papers that only says what the sources support.**

- Routes each question to the right document using a cross-encoder against each paper's title and abstract, so sentences from the wrong paper can't leak in
- Reranks before narrowing, so the answer-bearing sentence is never dropped before the best relevance judge sees it
- Selects evidence with maximal marginal relevance using a relevance/redundancy trade-off that shifts as the word budget fills
- Answers in the same language as the question
- Scores faithfulness afterwards with a multilingual NLI model
- The ablation study showed the reranker matters most: removing it roughly halved faithfulness (99.3% → 49.9%)

`Python` `sentence-transformers` `mDeBERTa NLI` `scikit-learn` `OpenAI / Ollama`

---

## 📊 [data-scaling-reliability](https://github.com/SleezyBear/data-scaling-reliability)

**Does more training data always help? An empirical study of where it stops paying off.**

- Seven classifiers (logistic regression to MLP) on three real datasets, including heavily imbalanced fraud data
- Training size scaled from 20% to 100%, with label noise, feature noise and near-duplicate "weak" samples injected at increasing levels
- Wilcoxon and Friedman tests with Holm and Benjamini–Hochberg corrections, so the conclusions survive multiple comparisons
- Automatic detection of saturation points, noise-tolerance limits and inflection points in the scaling curves
- Fully deterministic, with per-condition seeds, and resumable from checkpoints

`Python` `scikit-learn` `SciPy` `pandas` `matplotlib`

---

## 📄 [offline-image-to-pdf](https://github.com/SleezyBear/offline-image-to-pdf)

**A Flutter app that turns photos into PDFs without ever touching the network.**

- Capture pages with the camera, crop them and adjust brightness and contrast
- Drag to reorder pages, then generate the PDF on the device
- Keep a library of documents, and lock any of them behind a password stored only as a hash

`Flutter` `Dart`

---

## 📫 Contact

[madhav.nanda@gmail.com](mailto:madhav.nanda@gmail.com)
