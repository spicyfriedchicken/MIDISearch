# MIDISearch (Real-Time Classical Music Recognition)

Most commercial audio fingerprinting systems like Shazam rely on spectral hashing or robust fingerprinting of short audio clips. MidiSearch takes a different approach: it leverages learned neural embeddings of MIDI or audio segments and performs **real-time vector search** across a database of classical pieces to identify the closest match.

---

# MIDISearch: Neural Search for Classical MIDI Music

MidiSearch is a distributed, GPU-accelerated music recognition system that accepts incoming MIDI or audio clips, generates dense embeddings using a trained model, and performs real-time nearest neighbor search across a large corpus of classical music. Built for low-latency inference and high-accuracy retrieval, it enables music identification using short performance segments.

This README outlines the architecture and design decisions for MidiSearch, modeled after scalable ML inference systems with optimizations for concurrency, latency, and distributed inference.

---

## Top-Level Architecture

```text
      Clients
        ↓
   [gRPC Router(s)]  ←→  [HNSW/FAISS Index + Metadata Store]
        ↓
   [ZeroMQ Worker Pool (Embedding Inference)]
        ↓
   [ONNX Embedding Model → Vector Matching]
```

---

## Client Node

The Client is the user-facing entry point into MidiSearch. It captures a short MIDI or audio clip, packages it into a Protobuf message with metadata (e.g., duration, tempo, input type), and sends it to the Router.

### Responsibilities

- Capture or accept input (live MIDI, audio, file)
- Preprocess (optional CQT or MIDI tokenization)
- Encode input via Protobuf
- Attach metadata (clip length, stream vs file, etc.)
- Send request via **gRPC**
- Receive top-k match result(s)
- Optionally visualize or play preview

---

## Client <-> Router Communication Layer

This layer acts as the frontend API for the system. It validates requests, attaches tracing metadata, and dispatches jobs internally.

### Why gRPC?

| Feature                  | gRPC Benefit                                |
| ------------------------ | ------------------------------------------- |
| **Binary input support** | Protobuf for efficient compact encoding     |
| **Streaming support**    | Full-duplex for live input streams          |
| **Timeouts & retries**   | Built-in deadline and retry semantics       |
| **Observability**        | OpenTelemetry interceptors, structured logs |

Unlike REST or raw TCP, gRPC balances performance with structured contract-based APIs, ideal for downstream orchestration.

---

## Router Node

The Router handles validated requests, determines worker routing, enforces limits, and acts as a reply proxy. It does **not** run any inference itself.

### Responsibilities

- Accept gRPC request
- Parse and validate metadata
- Generate `JobID`
- Dispatch via **ZeroMQ ROUTER socket** to backend workers
- Track active jobs in `unordered_map<JobID, Context>`
- Handle response/error propagation
- Log latency, load, and match outcomes

> The Router is horizontally scalable and stateless except for active job tracking.

---

## Router <-> Worker Communication Layer

Uses **ZeroMQ** for ultra-fast asynchronous communication between Router and Worker nodes.

### Benefits:

- DEALER/ROUTER pattern supports multiple in-flight jobs
- Extremely low overhead and flexible routing policies
- Easier batching, retries, and worker-level load balancing

### What We Build on Top of ZeroMQ:

- Protobuf-over-ZeroMQ envelope:
  ```
  JobID | ModelName | SerializedInput | Metadata
  ```
- Job correlation map
- Timeout + retry logic
- Health check pings
- Error codes and failure metadata

---

## Worker Node (Inference Layer)

Each Worker is a process bound to a GPU or CPU that handles:

### Responsibilities

- Connect to Router via ZeroMQ DEALER
- Receive and parse jobs
- Preprocess input (`MIDI → tensor`, or `Audio → CQT`)
- Load or reuse ONNX model via **ONNXRuntime**
- Run inference → embedding
- Normalize and search via **HNSWlib** or **FAISS**
- Return top-k match with confidence scores
- Optionally send health updates (heartbeat)

### Tasks to Implement

| Task                    | Description                      |
| ----------------------- | -------------------------------- |
| `ZmqWorkerContext`      | Setup ZMQ sockets                |
| `receive_and_parse_job` | Deserialize Protobuf             |
| `execute_embedding()`   | Convert clip → embedding         |
| `run_vector_search()`   | HNSWlib/FAISS similarity search  |
| `send_result()`         | Return Protobuf-encoded result   |
| `handle_error()`        | Send structured failure response |
| `heartbeat_loop()`      | (Optional) periodic health info  |

---

## Worker Node (Data Layer)

The Data Layer runs the core inference logic:

| Task                        | Description                                           |
| --------------------------- | ----------------------------------------------------- |
| `load_onnx_model()`         | Load DMET ONNX model from disk                        |
| `allocate_buffers()`        | Allocate input/output tensors                         |
| `preprocess_clip()`         | CQT/MIDI tokenization + normalization                 |
| `run_inference()`           | ONNXRuntime inference call                            |
| `postprocess_embedding()`   | Normalize + L2 vector                                 |
| `run_vector_search()`       | Approximate top-k nearest neighbors from vector index |
| `benchmark_model_latency()` | Measure end-to-end latency per clip                   |

TensorRT is **not** used unless the model proves computationally heavy, in which case FP16 optimization can be added.

---

## Metadata Store

Once a match is found, it is mapped to song metadata for display.

- Sources: SQLite, Redis, or static JSON
- Includes title, composer, duration, match start time, optional preview

---

## Use Cases and Targets

| Mode            | Description                      | Latency Target |
| --------------- | -------------------------------- | -------------- |
| Clip Match      | Single 5–10s clip                | <150ms         |
| Live MIDI Match | Streamed input                   | <100ms         |
| Audio Match     | Audio → embedding (CQT pipeline) | <300ms         |

---

## Deployment & Monitoring

- Dockerized microservices
- Observability: OpenTelemetry + Prometheus
- Metrics: QPS, latency, worker health, match accuracy
- CI: Unit + integration tests on gRPC + Worker logic

---

## TODO

-

---

## Special Thanks

To the creators of the [MAESTRO](https://magenta.tensorflow.org/datasets/maestro) dataset, and the open-source music ML community for tools like HNSWlib, FAISS, and ONNX Runtime.

