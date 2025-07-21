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

## Client Node 

The Client is the user-facing entry point into the inference system. It prepares the image/audio/video input, encodes it according to a communication protocol (e.g., Protobuf for gRPC), and sends the request to the Router along with metadata such as the model to run, desired parameters, and any routing hints.

### Responsibilities:

    Accept user input (image, audio, etc.)

    Encode input for transmission (Protobuf, byte array)

    Attach inference metadata (e.g., model type, resolution, mode)

    Send request to the Router via gRPC

    Optionally stream input or receive streamed output

    Display or save results returned from the server

## Client < -- > Router Communication Layer, Communication Protocol

The Client <-> Router Communication Layer is the frontend API layer where the Client Node (CLI) sends an image, audio, or request payload to the ML inference system, and the router decides how to process it.

The main role of this layer is to:
- Act as the entry point to an inference system
- Validate, Enrich and forward requests to internal components
- Enforce rate limits, logging, and input formats

Within Client <-> Router communication, there are four main options that I am considering:

### REST (HTTP)

REST is easy to test and integrate but incurs excessive overhead for binary data like images and lacks built-in streaming support, making it unsuitable for high-performance ML inference. It also requires clunky MIME handling and doesn't scale well for large or real-time inputs.

### TCP / ASIO
Raw TCP gives full control and excellent performance but demands manual implementation of framing, retries, timeouts, and error handling, making it extremely fragile and hard to maintain. It also lacks interoperability with ML tooling and doesn't support standard request semantics.

### gRPC (Selected)

gRPC offers efficient binary encoding via Protobuf, built-in streaming, and strong contracts with deadline/retry semantics, making it ideal for structured, scalable ML inference APIs. It strikes the right balance between performance, reliability, and developer ergonomics.

### ZeroMQ (Alternative)

ZeroMQ provides ultra-low latency and flexibility in communication patterns, making it a strong choice for internal high-frequency inference workloads. However, its lack of built-in schemas and reliability features makes it better suited for backend/internal layers than client-facing APIs.

### Comparing gRPC and ZeroMQ:

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

While both gRPC and ZeroMQ support binary data transfer, gRPC was chosen for the Client <-> Router layer because it offers structured, versioned APIs with first-class support for deadlines, retries, and streaming, making it significantly easier to build, maintain, and extend. Unlike ZeroMQ, gRPC provides a strong schema contract via .proto files, excellent language support, built-in observability (e.g., OpenTelemetry), and simple client integration without the need for manual connection or message framing logic. Although ZeroMQ excels in raw performance and flexibility, these advantages are better leveraged internally (e.g., Router <-> Worker).

## Router Node

The Router is the central coordinator that receives validated inference requests from the client and dispatches them to appropriate worker nodes. It parses the request, determines routing logic (e.g., which model or GPU to use), manages timeouts, and serves as the reply gateway back to the client. It does not run inference itself.

### Responsibilities:

    Receive and parse gRPC request

    Validate payload and metadata (e.g., file format, model exists)

    Enforce rate limits and logging

    Determine appropriate worker route (by load, task, or GPU availability)

    Serialize and dispatch job to a worker via ZeroMQ

    Track job status and collect result

    Return result to client via gRPC response or stream

## Router <-> Worker Communication Layer, Communication Protocol

The Router <-> Worker Communication Layer is the internal coordination layer where the router distributes inference jobs to available workers for execution on a GPU. This layer is responsible for efficiently routing jobs, balancing load, and ensuring low-latency communication between processes or machines within the inference system.
    
Unlike the pevious layer, Component B uses ZeroMQ to efficiently dispatch jobs from the Router to Workers with near-zero overhead, flexible messaging patterns, and full control over batching and scheduling.
gRPC is heavyweight and restrictive here, whereas ZeroMQ is ideal for internal, high-throughput, low-latency inference coordination. (Honestly, gRPC is fine and simpler to use here, I just want to learn ZeroMQ)

What must be built on top of 0mq:

- Message Framing & Protocol Schema (i.e JobID | ModelName | SerializedInput | Metadata )
    - Protobuf over ZeroMQ
- Routing and Load Balancing (Round Robin, Least-Loaded, etc)
- Job Tracking & Correlation (Track Jobs via JobID)
    - Likely use std::unordered_map<JobID, Context>
- Timeouts, Retry & Health Checks
- Erorr Propagation

## Worker Node (Communication Layer)

The Worker Node is a dedicated process responsible for executing inference jobs it receives from the router. It connects to the router over ZeroMQ, listens for incoming tasks, runs the appropriate model on local hardware (Likely GPU), and sends the results back to the router. Workers are model-aware and are specialized to a specific GPU.

### Responsibilities:

    Connect to the router via ZeroMQ (e.g., PULL or DEALER socket)

    Parse the incoming job (deserialize Protobuf payload)

    Load or reuse the appropriate model (e.g., SuperRes, Denoise, Transcribe)

    Run the model inference on a local device (typically a GPU)

    Return results (e.g., enhanced image, audio transcript) back to the router

    Optionally respond with error codes or structured failure metadata

    Optionally expose health signals (e.g., heartbeats, GPU usage, capacity)

### Tasks to implement:

| Module                          | Description                                      |
| ------------------------------- | ------------------------------------------------ |
| `ZmqWorkerContext`              | Initializes and manages the ZMQ socket           |
| `receive_and_parse_job()`       | Blocking receive + Protobuf decode               |
| `execute_job_stub()`            | Simulated inference logic                        |
| `send_result()`                 | Protobuf encode and send response                |
| `handle_error()`                | Catch + respond with structured error            |
| `heartbeat_loop()` *(optional)* | Background health signal or availability updates |
| `signal_handler()` *(optional)* | Graceful exit on shutdown signals                |

## Worker Node (Data Layer)

The Data Layer of the Worker Node handles everything from loading the trained SRCNN model (exported from PyTorch via ONNX) to executing optimized GPU inference using TensorRT. It preprocesses raw image bytes into normalized tensors, runs the super-resolution model on ImageNet-scale inputs using CUDA-accelerated execution, and postprocesses the output into final image formats. This layer ensures the inference pipeline is both accurate and high-performance, leveraging FP16 or INT8 optimizations where possible.

### Tasks to implement:

| Module                               | Description                                                           |
| ------------------------------------ | --------------------------------------------------------------------- |
| `load_onnx_model()`                  | Load exported ONNX model from disk and parse it using TensorRT APIs   |
| `build_trt_engine()`                 | Build a serialized TensorRT engine from ONNX (with FP16/INT8 support) |
| `allocate_trt_buffers()`             | Allocate GPU input/output buffers for inference                       |
| `preprocess_image()`                 | Convert image bytes → normalized float tensor (NCHW, resized, padded) |
| `run_trt_inference()`                | Launch inference using TensorRT execution context                     |
| `postprocess_output()`               | Convert output tensor → image byte format (e.g., uint8 PNG/JPEG)      |
| `optimize_model_fp16()` *(optional)* | Use builder flags to convert ONNX model to FP16-optimized engine      |
| `optimize_model_int8()` *(optional)* | Use TensorRT INT8 calibration + representative dataset for max perf   |
| `cache_engine()` *(optional)*        | Serialize and store `.engine` binary to avoid rebuilding on startup   |
| `benchmark_model_latency()`          | Measure model throughput (fps) and latency (ms/frame) on target GPU   |
