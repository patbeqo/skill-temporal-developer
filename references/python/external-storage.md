# Python SDK External Storage

> [!NOTE]
> This feature is in Public Preview. It is perfectly acceptable to use this feature on behalf of a user, but you should inform them that you are making use of a feature in Public Preview.

## What this is

External Storage uses the **claim check pattern**: it offloads each Payload to an external store (e.g. Amazon S3), records a small reference token (the "claim check") in Event History, and uses that token to retrieve the Payload when needed. The SDK handles storage and retrieval transparently.

## When to use it

- A Workflow input, Activity input, Activity result, or Workflow result will exceed the **2 MB** per-payload limit (fixed at 2 MB on Temporal Cloud; configurable on self-hosted only).
- Long Event Histories degrade Workflow Task latency (e.g. AI agent conversations growing per turn).
- The user wants payload data to live in storage **they** control. Set `payload_size_threshold=0` to externalize all payloads.
- The user is migrating from self-hosted (with a larger configured limit) to Temporal Cloud.

## Where it sits in the pipeline

Order: **Payload Converter → Payload Codec → External Storage**. Storage runs last on outbound; it reverses on inbound.

Consequences:

- If a Payload Codec encrypts data, the bytes are already encrypted **before** upload.
- The Temporal UI displays the reference token, not the data; the SDK retrieves the payload transparently before handing it to your Workflow or Client.

## Concurrency

The SDK uploads and downloads payloads **concurrently** within a single Workflow Task — multiple offloaded payloads in one Task are stored or retrieved in parallel, not sequentially. This is automatic.

## Setup with the built-in S3 driver

Install the `aioboto3` extra:

```bash
python -m pip install "temporalio[aioboto3]"
```

Create the driver, attach it to a `DataConverter`, and pass the converter to both Client and Worker:

```python
import aioboto3
import dataclasses
from temporalio.client import Client, ClientConfig
from temporalio.contrib.aws.s3driver import S3StorageDriver
from temporalio.contrib.aws.s3driver.aioboto3 import new_aioboto3_client
from temporalio.converter import DataConverter, ExternalStorage
from temporalio.worker import Worker

session = aioboto3.Session(profile_name=AWS_PROFILE, region_name=AWS_REGION)
async with session.client("s3") as s3_client:
    driver = S3StorageDriver(
        client=new_aioboto3_client(s3_client),
        bucket="my-temporal-payloads",
    )

    data_converter = dataclasses.replace(
        DataConverter.default,
        external_storage=ExternalStorage(drivers=[driver]),
    )

    client_config = ClientConfig.load_client_connect_config()
    client = await Client.connect(**client_config, data_converter=data_converter)

    worker = Worker(
        client,
        task_queue="my-task-queue",
        workflows=[],
        activities=[],
    )
```

The S3 driver uses standard AWS credentials from the environment (env vars, IAM role, or AWS config file).

Workflows and Activities on the Worker use the driver automatically — no business-logic changes.

## Payload size threshold

- Default: **256 KiB**.
- Set `payload_size_threshold=0` to externalize **all** payloads regardless of size.

```python
data_converter = dataclasses.replace(
    DataConverter.default,
    external_storage=ExternalStorage(
        drivers=[driver],
        payload_size_threshold=0,
    ),
)
```

## Multiple drivers and migration

When you register more than one driver, you **must** supply a `driver_selector` function. The selector chooses which driver stores each payload. Unselected drivers remain available for **retrieval** — this is how you migrate between storage backends without losing access to existing claims.

- Return `None` from the selector to keep a specific payload inline in Event History.

```python
preferred_driver = S3StorageDriver(
    client=new_aioboto3_client(s3_client),
    bucket="my-bucket",
)
legacy_driver = LegacyStorageDriver()

ExternalStorage(
    drivers=[preferred_driver, legacy_driver],
    driver_selector=lambda context, payload: preferred_driver,
)
```

## Custom storage driver

Extend `StorageDriver` and implement **three** methods:

- `name() -> str` — unique identifier for the driver, stored in the claim reference so the SDK can route retrieval. Renaming after payloads are stored **breaks retrieval**.
- `async store(context, payloads) -> list[StorageDriverClaim]` — upload each Payload and return one claim per payload. A claim is a `dict[str, str]` the driver uses to locate the payload later.
- `async retrieve(context, claims) -> list[Payload]` — download bytes using claim data and reconstruct each Payload.

Inside `store()`, serialize each payload with `payload.SerializeToString()`; in `retrieve()`, reconstruct with `payload.ParseFromString(data)`. The application data has already been serialized by the Payload Converter and Payload Codec before reaching the driver.

`context.target` provides identity information (namespace, Workflow ID, or Activity ID). Check the target type with `isinstance(target, StorageDriverWorkflowInfo)`; the Workflow info exposes `target.namespace` and `target.id`. Use this to scope storage keys per Workflow.

Worked example — local-disk driver (development/testing only):

```python
import os
import uuid
from typing import Sequence

from temporalio.api.common.v1 import Payload
from temporalio.converter import (
    StorageDriver,
    StorageDriverClaim,
    StorageDriverRetrieveContext,
    StorageDriverStoreContext,
    StorageDriverWorkflowInfo,
)


class LocalDiskStorageDriver(StorageDriver):
    def __init__(self, store_dir: str = "/tmp/temporal-payload-store") -> None:
        self._store_dir = store_dir

    def name(self) -> str:
        return "local-disk"

    async def store(
        self,
        context: StorageDriverStoreContext,
        payloads: Sequence[Payload],
    ) -> list[StorageDriverClaim]:
        os.makedirs(self._store_dir, exist_ok=True)

        prefix = self._store_dir
        target = context.target
        if isinstance(target, StorageDriverWorkflowInfo) and target.id:
            prefix = os.path.join(self._store_dir, target.namespace, target.id)
            os.makedirs(prefix, exist_ok=True)

        claims = []
        for payload in payloads:
            key = f"{uuid.uuid4()}.bin"
            file_path = os.path.join(prefix, key)
            with open(file_path, "wb") as f:
                f.write(payload.SerializeToString())
            claims.append(StorageDriverClaim(claim_data={"path": file_path}))
        return claims

    async def retrieve(
        self,
        context: StorageDriverRetrieveContext,
        claims: Sequence[StorageDriverClaim],
    ) -> list[Payload]:
        payloads = []
        for claim in claims:
            file_path = claim.claim_data["path"]
            with open(file_path, "rb") as f:
                raw = f.read()
            payload = Payload()
            payload.ParseFromString(raw)
            payloads.append(payload)
        return payloads
```

Wire the custom driver into the Data Converter the same way as the S3 driver:

```python
data_converter = dataclasses.replace(
    DataConverter.default,
    external_storage=ExternalStorage(
        drivers=[LocalDiskStorageDriver()],
    ),
)
```

You can package a custom driver as a [plugin](/develop/plugins-guide) for reuse across services.

## Codec Server with External Storage

When Workers and Clients use External Storage, Event History contains reference tokens — not payload data. For the Web UI and CLI to show decoded payloads, the Codec Server must download from external storage **and** decode through the Payload Codec in the correct order.

Build the Codec Server with a payload HTTP handler that accepts your storage drivers, your pre-storage codecs (the Payload Codecs your Workers use), and any post-storage codecs (applied by a proxy after external storage). The handler applies them in the correct order across all endpoints.

Endpoints exposed when storage drivers are configured:

- **`/download`** — retrieves payload data from external storage and decodes it through the Payload Codec. The Web UI calls this when a user clicks to view the full payload behind a reference.
- **`/decode`** — decodes encoded payloads and, by default, retrieves storage references inline. Pass `?preserveStorageRefs=true` to return storage references as-is without retrieval.
- **`/encode`** — applies the Payload Codec, then uploads payloads exceeding the threshold and replaces them with reference tokens.

**Don't point a Worker's remote codec at the storage-aware handler** — it runs the full encode-store-encode and decode-retrieve-decode pipeline. Run a separate non-storage codec HTTP handler for remote codecs, configured with the same codecs.

The [Python External Storage sample](https://github.com/temporalio/samples-python/tree/main/external_storage) demonstrates a storage-aware Codec Server implementation.

## Lifecycle management

Temporal does **not** auto-delete payloads from your store. Configure a TTL on your bucket:

```
TTL > Maximum Workflow Run Timeout + Namespace Retention Period
```

Example: Run Timeout 14 days + Namespace retention 30 days → set TTL to at least 44 days.

For Workflows with no finite Run Timeout, there is no safe finite TTL. Use Continue-as-New so the new run uploads fresh payloads and the old run's payloads only need to survive its retention period.

## Anti-patterns

- **Don't change the value returned by `name()` after payloads have been stored.** The name is embedded in the claim reference; renaming breaks retrieval of existing claims.
- **Don't use `payload_size_threshold=1` to mean "externalize all"** — use `payload_size_threshold=0`. (This sentinel differs from Go, where `0` is the default and `1` externalizes all.)
- **Don't register multiple drivers without a `driver_selector`.** The selector is required when there is more than one driver.
- **Don't pass the storage-aware payload HTTP handler as a Worker's remote codec target.** Use a separate non-storage codec HTTP handler for that role.
- **Don't omit a TTL on the bucket.** Payloads can be orphaned if a request fails after upload.
