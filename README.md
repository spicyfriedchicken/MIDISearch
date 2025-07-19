# ML Inference Server

The ML Inference Server receives image inputs from clients and routes them efficiently to available GPU workers for processing. It is optimized for low-latency, high-throughput inference using gRPC for structured client communication and ZeroMQ for fast internal dispatch. The system supports batching, streaming, and intelligent load balancing across workers to deliver scalable and reliable ML model serving. This project is currently in the planning stage, the associated README outlines the architecture and decisions made with their rationales.

## Top - Level Architecture

    Clients
       ↓
    [Router(s)]  ←→  [Raft Cluster for Metadata]
       ↓
    [Inference Workers]
       ↓
    [TensorRT Inference]

## Component A: Client < -- > Router Communication Layer, Communication Protocol

The Client <-> Router Communication Layer is the frontend API layer where the client (CLI) sends an image, audio, or request payload to the ML inference system, and the router decides how to process it.

The main role of this layer is to:
- Act as the entry point to an inference system
- Validate, Enrich and forward requests to internal components
- Enforce rate limits, logging, and input formats

Within Client <-> Router communication, there are four main options that I am considering:

1. REST Protcol (HTTP)
2. gRPC
3. ZeroMQ
4. Raw TCP / Boost.Asio 

| Option              | Pros                                                                                                                                                                                                                             | Cons                                                                                                                                                                          |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **REST (HTTP)**     | - Universally supported for curl-based testing<br>- JSON body for structured input<br>- Easy integration with most inference SDKs (e.g., Python requests, Postman)                                                                                                       | - String-based payloads inefficient for binary data (e.g., images)<br>- No built-in streaming (can’t stream large inputs/outputs efficiently)<br>- Needs chunking & MIME handling for large files                            |
| **gRPC (Selected)** | - Binary encoding (Protobuf) ideal for tensors, images, embeddings<br>- Built-in support for deadlines, retries, status codes<br>- Full-duplex streaming → stream input frames / batched requests<br>- HTTP/2 multiplexing → multiple inference requests over 1 connection | - Requires clients to use generated stubs<br>- Not human-readable for quick CLI testing<br>- More complex to debug payloads manually (no curl/wireshark inspection)                                                          |
| **ZeroMQ**          | - Extremely low-latency<br>- Efficient for high-frequency inference workloads<br>- Suited for REQ/REP (inference call/response) or DEALER/ROUTER (load-balanced)<br>- No framing overhead or HTTP state                                                                    | - No standardized schema enforcement (e.g., no Protobuf unless layered on)<br>- You must implement timeout, retry, error propagation manually<br>- Requires custom message routing on client side                            |
| **Raw TCP / ASIO**  | - Ultimate control over framing, compression, deadlines<br>- Optimal throughput for raw binary inputs (e.g., raw tensor buffers)<br>- Ideal for extremely latency-sensitive inference                                                                                    | - Extremely error-prone: must implement framing, retries, timeouts, routing, and backpressure from scratch<br>No standard for request semantics or message structure<br> - Not interoperable with ML clients out-of-the-box |

This leaves us with only two _real_ options for a image/video/audio based inference server - **gRPC** and **ZeroMQ**. Let's compare each of these:

| Criterion                           | **gRPC**                                                                               | **ZeroMQ**                                                                                      |
| ----------------------------------- | -------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **Binary data support**             | Encodes via Protobuf; good for small-to-medium images (e.g., ≤ 1MB); base64 not needed | Sends raw bytes directly; ideal for larger images and serialized tensors                        |
| **Structured request contracts**    | Strongly typed via `.proto`; easy to version APIs                                      | None built-in; must define your own protocol or wrap in Protobuf manually                       |
| **Deadline + retry semantics**      | Built-in: `context.WithTimeout()`, retry interceptors                                  | Must be implemented manually; no backoff or error classification                                |
| **Streaming support**               | Full-duplex via HTTP/2; can stream video frames or result streams                      | Supports PUSH/PULL, PUB/SUB, DEALER/ROUTER patterns; powerful but must manually manage sessions |
| **Ecosystem/tooling**               | Excellent in Go/Python/C++; observability, TLS, interceptors                           | Lightweight, but no observability, tracing, or pluggable middleware                             |
| **Ease of client implementation**   | Simple: `import grpc`, generate stub, call RPC                                         | Needs message envelope design, connection loop, and state mgmt                                  |
| **Performance for images/videos**   | Very good, but you’ll need to avoid massive message sizes (>4MB Protobufs = perf hits) | Excellent; handles raw frames, even real-time video or tensor data streams                      |
| **Maintainability**                 | `.proto` acts as a clean schema contract; easy for other teams to consume              | Protocol drift risk unless you formalize a schema (e.g., Protobuf over ZeroMQ)                  |
| **Extensibility for observability** | First-class support (e.g., OpenTelemetry interceptors, logging middleware)             | Must be done manually                                                                           |

