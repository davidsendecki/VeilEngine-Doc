# Asset System

The runtime **AssetSystem** owns loaded asset records and coordinates dependency batches used during world and resource loading.

## Core responsibilities

`CAssetSystem` currently provides model requests, dependency-batch management, synchronous batch loading, progress queries, model state queries, and read-only access to ready model data.

Asset requests are deduplicated by path. Requesting an already-known model therefore resolves to the existing asset handle rather than creating another independent record.

## Asset handles

Runtime assets are addressed through typed `AssetHandle<T>` values. This keeps references lightweight and avoids exposing subsystem-owned storage directly to callers.

The AssetSystem retains ownership of the actual asset memory. Views returned to callers are therefore non-owning and remain valid only while the corresponding data remains loaded and the AssetSystem remains alive.

## Dependency batches

A dependency batch groups assets required by a larger loading operation. The current model workflow is:

```text
CreateDependencyBatch
        │
        ├─ RequestModel(..., Batch)
        ├─ RequestModel(..., Batch)
        │
        ▼
LoadDependencyBatch
        │
        ▼
GetDependencyBatchProgress
        │
        ▼
DestroyDependencyBatch
```

Destroying a batch removes its dependency bookkeeping; it does **not** unload the assets referenced by that batch.

## Loading and fallback behavior

Models begin as requests and are loaded when their dependency batch is processed. Assets that are already ready are skipped.

The AssetSystem also owns a required fallback/error model. If an ordinary model fails to load but the fallback is available, the failed request can resolve through that fallback and report a ready state to consumers. Failure of the fallback asset itself is treated differently because no valid substitute then exists.

## Current scope

The implementation is currently strongly model-oriented. The architecture leaves room for additional asset classes to adopt the same handle/storage/loading model as the asset pipeline grows.

!!! warning "Lifetime rule"
    Do not retain raw pointers into AssetSystem-owned model data beyond the lifetime guaranteed by the AssetSystem. Handles are the durable reference mechanism; resolved views are borrowed data.
