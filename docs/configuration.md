# Configuration

This document outlines the environment variables available for configuring the `worker-comfyui`.

## General Configuration

| Environment Variable | Description                                                                                                                                                                                                                  | Default |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------- |
| `REFRESH_WORKER`     | When `true`, the worker pod will stop after each completed job to ensure a clean state for the next job. See the [RunPod documentation](https://docs.runpod.io/docs/handler-additional-controls#refresh-worker) for details. | `false` |
| `SERVE_API_LOCALLY`  | When `true`, enables a local HTTP server simulating the RunPod environment for development and testing. See the [Development Guide](development.md#local-api) for more details.                                              | `false` |
| `COMFY_ORG_API_KEY`  | Comfy.org API key to enable ComfyUI API Nodes. If set, it is sent with each workflow; clients can override per request via `input.api_key_comfy_org`.                                                                        | –       |

## Logging Configuration

| Environment Variable | Description                                                                                                                                                      | Default |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------- |
| `COMFY_LOG_LEVEL`    | Controls ComfyUI's internal logging verbosity. Options: `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`. Use `DEBUG` for troubleshooting, `INFO` for production. | `DEBUG` |

## Debugging Configuration

| Environment Variable           | Description                                                                                                            | Default |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------- | ------- |
| `WEBSOCKET_RECONNECT_ATTEMPTS` | Number of websocket reconnection attempts when connection drops during job execution.                                  | `5`     |
| `WEBSOCKET_RECONNECT_DELAY_S`  | Delay in seconds between websocket reconnection attempts.                                                              | `3`     |
| `WEBSOCKET_TRACE`              | Enable low-level websocket frame tracing for protocol debugging. Set to `true` only when diagnosing connection issues. | `false` |

> [!TIP] > **For troubleshooting:** Set `COMFY_LOG_LEVEL=DEBUG` to get detailed logs when ComfyUI crashes or behaves unexpectedly. This helps identify the exact point of failure in your workflows.

## S3-Compatible Storage Upload Configuration

Configure these variables **only** if you want the worker to upload generated images directly to an S3-compatible storage service (AWS S3 or Cloudflare R2). If these are not set, images will be returned as base64-encoded strings in the API response.

### AWS S3 Configuration

- **Prerequisites:**
  - An AWS S3 bucket in your desired region.
  - An AWS IAM user with programmatic access (Access Key ID and Secret Access Key).
  - Permissions attached to the IAM user allowing `s3:PutObject` (and potentially `s3:PutObjectAcl` if you need specific ACLs) on the target bucket.

### Cloudflare R2 Configuration

- **Prerequisites:**
  - A Cloudflare R2 bucket.
  - An R2 API token with read and write permissions. Create one in the Cloudflare dashboard under **R2 > Manage R2 API Tokens**.
  - Your Cloudflare Account ID (found in the R2 dashboard URL or account settings).

| Environment Variable       | Description                                                                                                                             | Example (AWS S3)                                                    | Example (Cloudflare R2)                                    |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- | ---------------------------------------------------------- |
| `BUCKET_ENDPOINT_URL`      | The full endpoint URL of your storage bucket. **Must be set to enable upload.**                                                         | `https://<your-bucket-name>.s3.<aws-region>.amazonaws.com`         | `https://<account-id>.r2.cloudflarestorage.com`           |
| `BUCKET_ACCESS_KEY_ID`     | Your access key ID. Required if `BUCKET_ENDPOINT_URL` is set.                                                                           | `AKIAIOSFODNN7EXAMPLE`                                             | Your R2 API token Access Key ID                             |
| `BUCKET_SECRET_ACCESS_KEY` | Your secret access key. Required if `BUCKET_ENDPOINT_URL` is set.                                                                       | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY`                         | Your R2 API token Secret Access Key                        |
| `BUCKET_NAME`               | The name of your bucket. **Required for R2, optional for S3** (if not set, falls back to `rp_upload`).                                   | `my-s3-bucket`                                                      | `my-r2-bucket`                                             |
| `BUCKET_REGION`             | The AWS region (only used for S3, ignored for R2). Optional, defaults to `us-east-1`.                                                   | `us-east-1`                                                        | N/A (not used for R2)                                      |

**Note:** 
- If `BUCKET_NAME` is set, the worker will use a custom upload function that supports both AWS S3 and Cloudflare R2. This provides better R2 compatibility and allows you to use custom R2 endpoints.
- If `BUCKET_NAME` is not set, the worker will fall back to the `runpod` Python library helper `rp_upload.upload_image`, which handles creating a unique path within the bucket based on the `job_id` but may not support R2 custom endpoints.

### Example Upload Response

If the storage environment variables are correctly configured, a successful job response will look similar to this:

```json
{
  "id": "sync-uuid-string",
  "status": "COMPLETED",
  "output": {
    "images": [
      {
        "filename": "ComfyUI_00001_.png",
        "type": "s3_url",
        "data": "https://your-bucket-name.s3.your-region.amazonaws.com/sync-uuid-string/ComfyUI_00001_.png"
      }
      // Additional images generated by the workflow would appear here
    ]
    // The "errors" key might be present here if non-fatal issues occurred
  },
  "delayTime": 123,
  "executionTime": 4567
}
```

The `data` field contains the URL to the uploaded image file in your storage bucket. The path usually includes the job ID.
