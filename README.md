
<div align="center">

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/openlake-project/openlake/refs/heads/main/assets/openlake-wordmark-dark-8192.png">
  <img alt="OpenLake" src="https://raw.githubusercontent.com/openlake-project/openlake/refs/heads/main/assets/openlake-wordmark-light-8192.png" width=55%>
</picture>


<h3 align="center">
The shortest path from NVMe to GPU memory.
</h3>

S3 wire compatible distributed object storage, written in Rust on `io_uring`, for the workloads that move terabytes between storage and GPUs.


[Discord](https://discord.gg/TNXqVSnP6x)&nbsp;·&nbsp;[Website](https://theopenlake.com)&nbsp;·&nbsp;[Comparison](https://theopenlake.com/compare.html)&nbsp;·&nbsp;[Architecture](#architecture)&nbsp;·&nbsp;[Quickstart](#quickstart)



[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)
[![Rust](https://img.shields.io/badge/rust-1.91%2B-orange.svg)](rust-toolchain.toml)
[![Status](https://img.shields.io/badge/status-early%20development-yellow.svg)](#project-status)
[![Web](https://img.shields.io/badge/web-theopenlake.com-1d4ed8.svg)](https://theopenlake.com)



</div>


---


## What is OpenLake?

OpenLake is an object store for AI infrastructure. Training and inference clusters spend a large fraction of their wall clock time moving bytes from storage into GPU memory, and most object stores put the host CPU, the page cache, and several userspace copies directly in that path. OpenLake is a clean room, S3 wire compatible implementation that takes the opposite stance:

- **`io_uring`, thread per core.** Built on the [`compio`](https://github.com/compio-rs/compio) completion based runtime. One runtime per core, pinned, no work stealing. The HTTP frontend and the storage engine run on the *same* thread, so a request never crosses a core boundary on the hot path.
- **No CPU detour on the data path.** The design goal: GPUDirect Storage and RDMA so bytes flow NVMe → NIC → peer NIC → GPU VRAM without staging through host memory or the page cache. Not built yet; see [Architecture](#architecture).
- **MinIO compatible on disk.** Objects are laid out in the `xl.meta` format, so an existing MinIO or RustFS deployment's disk layout is intelligible to OpenLake, and vice versa.
- **Erasure coded, distributed.** SIMD Reed Solomon, fixed size erasure sets, deterministic placement. The durability model operators already know, with a much smaller runtime underneath it.

Today OpenLake runs as a standard S3 endpoint you can point any AWS SDK at.

## Key features

| | |
|---|---|
| **S3 wire compatible** | SigV4 authentication; bucket and object CRUD; batch delete; `ListObjects` v1 and v2; multipart upload; multi version objects. Works with the AWS CLI, `boto3`, the `aws-sdk-*` crates, `mc`, and other S3 clients. |
| **`io_uring` runtime** | [`compio`](https://github.com/compio-rs/compio) plus [`cyper`](https://github.com/compio-rs/cyper) / `cyper-axum`: hyper's HTTP/1.1 (and HTTP/2 for the cluster plane) on a completion based runtime. One pinned runtime per CPU, `SO_REUSEPORT` listeners, no tokio runtime spun up. |
| **SIMD erasure coding** | [`reed-solomon-simd`](https://crates.io/crates/reed-solomon-simd) (FFT algorithm; SSSE3 / AVX2 / NEON auto detected). Shards are streamed stripe by stripe, so peak RAM per in flight PUT is one stripe, not the whole object. |
| **MinIO `xl.meta` layout** | v1.x metadata format. Objects up to 128 KiB are inlined directly into `xl.meta`; larger ones are written as Reed Solomon shards across the set. |
| **Distributed by erasure sets** | A flat pool of disks is partitioned into fixed width sets at startup; every `(bucket, key)` hashes (SipHash) to exactly one set; write all, read any quorum within the set. Operators shape the failure profile by ordering nodes and choosing the set width and parity count. |
| **mTLS HTTP/2 cluster plane** | Every node is both a client and a server on the inter node RPC plane; HTTP/2 negotiated over mutual TLS (required for any cluster of more than one node). |
| **Distributed locking** | A `dsync` style lock service serializes multipart and metadata mutations across nodes. |
| **One static binary** | `phenomenald` (the storage node) and `phenomenal` (a local diagnostic and benchmark CLI). No external coordinator, no JVM, no GC. |

## Architecture

A request today takes this path:

```
S3 client ──HTTP──▶ cyper-axum  (on compio / io_uring)
                       │  SigV4 verify
                       ▼
                     Engine  ──▶  erasure set  ─┬─▶  local disk   (phenomenal_io, io_uring)
                       │     (SipHash route)     └─▶  peer node    (HTTP/2 + mTLS RPC)
                       ▼
                    xl.meta  +  Reed Solomon shards   /   inlined body
```

The data path we are building toward, where the CPU is not in the loop:

```
NVMe ──io_uring──▶ NIC ──RDMA · RoCEv2──▶ NIC ──GPUDirect──▶ GPU memory ──decompress──▶ CUDA kernel
```

### Workspace

```
crates/
├── phenomenal_io/        local FS I/O on io_uring · xl.meta encode/decode · on disk layout · RPC backend client
├── phenomenal_storage/   the engine · erasure coding · cluster topology and set routing · dsync locking · put/get/list/multipart
├── phenomenal_server/    S3 HTTP frontend (cyper-axum on compio) · SigV4 · inter node RPC server · lock server   →  `phenomenald`
└── phenomenal_cli/       local diagnostic and microbenchmark client (drives a LocalFsBackend directly)            →  `phenomenal`
```

> The crate namespace is still `phenomenal_*`: *OpenLake* is the project name, `phenomenal` was the working codename and remains the crate and binary prefix for now.

## Quickstart

Requires **Rust 1.91+** (pinned in `rust-toolchain.toml`). Linux gives you the `io_uring` driver; macOS builds and runs on the `kqueue` driver for development.

```sh
git clone <repo-url> openlake && cd openlake
cargo build --release --workspace
```

Write a node config. The full schema, and a multi node example, are documented at the top of [`crates/phenomenal_server/src/config.rs`](crates/phenomenal_server/src/config.rs):

```toml
# node0.toml: single node dev instance
self_id              = 0
data_dirs            = ["/var/lib/openlake/disk0", "/var/lib/openlake/disk1", "/var/lib/openlake/disk2"]
s3_addr              = "0.0.0.0:9000"
rpc_addr             = "0.0.0.0:9100"
set_drive_count      = 3
default_parity_count = 1            # EC[2+1] within the set: tolerates 1 disk loss
region               = "us-east-1"

[[credentials]]
access_key = "openlakeaccesskey"
secret_key = "openlakesecretkey"

[[nodes]]
id         = 0
rpc_addr   = "127.0.0.1:9100"
disk_count = 3
```

Run the node, then talk to it with any S3 client:

```sh
cargo run --release -p phenomenal_server -- --config node0.toml
# or:  ./target/release/phenomenald --config node0.toml

export AWS_ACCESS_KEY_ID=openlakeaccesskey
export AWS_SECRET_ACCESS_KEY=openlakesecretkey

aws --endpoint-url http://localhost:9000 s3 mb s3://demo
aws --endpoint-url http://localhost:9000 s3 cp ./checkpoint.safetensors s3://demo/
aws --endpoint-url http://localhost:9000 s3 ls s3://demo/
```

`phenomenal` (the CLI crate) is a **local** tool: it drives a `LocalFsBackend` directly for diagnostics and microbenchmarks (`phenomenal bench --n 100000 --size 4096`), not an S3 client.

## Contributing

Issues and pull requests are welcome. Before sending a PR:

```sh
cargo build  --workspace
cargo test   --workspace
cargo clippy --workspace --all-targets
cargo fmt    --all
```

`rustfmt` and `clippy` settings live in `.rustfmt.toml` and `rust-toolchain.toml`; the bar is a clean `clippy` and `fmt`.

## License

[Apache License 2.0](LICENSE).

---

<div align="center">
<sub><a href="https://theopenlake.com">theopenlake.com</a></sub>
</div>
