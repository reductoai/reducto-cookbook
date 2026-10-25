# r-1

Reducto r-1 is a page parser. You send one page image. The endpoint returns the layout blocks of the page, in reading order. Each block has a type, a box on the page, and its content. Tables come back as HTML.

This page describes the endpoint contract of version 3 of the r-1 model package on AWS Marketplace.

- [Access](#access)
- [Deploy](#deploy)
- [Request](#request)
- [Response](#response)
- [Errors](#errors)
- [Batch transform](#batch-transform)
- [Billing](#billing)
- [Limits](#limits)
- [Samples](#samples)

## Access

r-1 is available on AWS Marketplace through a private offer from Reducto.

1. Send Reducto the 12-digit ID of the AWS account that will run r-1. Reducto sends you a private offer link for that account.
2. Sign in to that AWS account, open the link, and accept the offer.
3. Open **AWS Marketplace > Manage subscriptions**, find r-1, and select **Configure**.
4. Choose version 3 or later, and a region that offers `ml.g6e.2xlarge` (for example `us-east-2`). Copy the model package ARN.

## Deploy

Deploy the model package to one `ml.g6e.2xlarge` instance, as a real-time endpoint or as a batch transform job. [`aws-sagemaker`](aws-sagemaker) has the steps and a notebook.

Delete the endpoint when you are done. AWS bills the instance while the endpoint exists.

## Request

Send one page per request. Call `InvokeEndpoint` with `Content-Type: application/json` and one of two bodies. The page must be a PNG or JPEG image.

### Page-parse body

```json
{"image_b64": "<base64 PNG or JPEG of one page>"}
```

Sample: [`samples/invoice.request.json`](samples/invoice.request.json).

### Chat-completions body

An OpenAI chat-completions request with one `user` message. The message has one `image_url` part, with a `data:` URL of a PNG or JPEG.

```json
{
  "model": "r-1",
  "messages": [
    {
      "role": "user",
      "content": [
        {"type": "image_url", "image_url": {"url": "data:image/png;base64,<base64 PNG>"}}
      ]
    }
  ]
}
```

Sample: [`samples/invoice.chat.request.json`](samples/invoice.chat.request.json).

r-1 ignores the `model` field and an optional second part of type `text`. The endpoint has no network access, so the image must be inline: remote image URLs do not work.

The chat body also accepts `stream: false`, `n: 1`, `user`, `seed`, and `stream_options`. These fields do not change the answer. Streaming (`stream: true`), `n` greater than 1, and tools return 422 `invalid request`.

### Optional fields

| Field | Body | Type | Notes |
|---|---|---|---|
| `max_pixels` | both | int | 1 to 6,291,456. Default 3,145,728. r-1 scales a larger image down to this number of pixels before it reads the page. |

## Response

### Page-parse body

```json
{
  "blocks": [
    {
      "type": "Title",
      "bbox": {"left": 0.056, "top": 0.111, "width": 0.256, "height": 0.036},
      "content": "INVOICE INV-314"
    },
    {
      "type": "Table",
      "bbox": {"left": 0.051, "top": 0.235, "width": 0.536, "height": 0.448},
      "content": "<table><tr><th>Item</th><th>Quantity</th><th>Amount</th></tr>...</table>"
    }
  ],
  "usage": {"prompt_tokens": 644, "completion_tokens": 235, "total_tokens": 879}
}
```

The full answer is [`samples/invoice.response.json`](samples/invoice.response.json).

### Chat-completions body

An OpenAI `chat.completion` object with `"model": "r-1"`. `choices[0].message.content` holds the same `{"blocks": [...]}` object as a JSON string, without `usage`. The token counts are in the top-level `usage` field. `finish_reason` is always `stop`.

The full answer is [`samples/invoice.chat.response.json`](samples/invoice.chat.response.json). To read the blocks in Python:

```python
blocks = json.loads(completion["choices"][0]["message"]["content"])["blocks"]
```

### Blocks

The blocks are in reading order. Each block has these fields:

| Field | Description |
|---|---|
| `type` | One of `Header`, `Footer`, `Title`, `Section Header`, `Page Number`, `List Item`, `Figure`, `Table`, `Key Value`, `Text`, `Comment`, `Signature`. |
| `bbox` | The box of the block: `left`, `top`, `width`, and `height` as fractions of the page width and height, from 0 to 1. `left` and `top` are the top-left corner. |
| `content` | The text of the block. A table is HTML. |

The block types and the `left`, `top`, `width`, and `height` box fields match the hosted Reducto Parse API. The box has no `page` or `original_page` field, and a block has no fields other than `type`, `bbox`, and `content`.

To get pixel positions, multiply by the size of the image that you sent:

```python
x = bbox["left"] * image_width
y = bbox["top"] * image_height
w = bbox["width"] * image_width
h = bbox["height"] * image_height
```

A response describes one page and has no page number. When you parse a multipage document, keep track of the page number of each request yourself.

A blank page can return an empty list or one block.

## Errors

| Status | `detail` | What to do |
|---|---|---|
| 413 | `request body too large` | Send a smaller image. The full JSON body must be at most 6 MB (6,291,456 bytes). In real time, SageMaker refuses a larger body before it reaches the endpoint. |
| 415 | `invalid input` | Send a readable PNG or JPEG of at most 25,165,824 pixels. |
| 400, 413, 422 | `invalid request` | Correct the request body. For example, a required field is missing, or the chat body has a `data:` URL that is not a PNG or JPEG or a field that is not accepted. The page-parse body ignores unknown fields. |
| 422 | `input could not be processed` | r-1 cannot process this page. Do not retry. See below. |
| 500, 502 | `inference failed` | Retry at most 2 times, with a delay between attempts. |
| 503 | `model is warming up` | Wait until the endpoint is ready, then retry. |
| 504 | `inference timed out` | Retry once. If the page times out again, skip it. |

In real time, SageMaker shows each error from the endpoint as a 424 `ModelError`. The status of the endpoint is in `OriginalStatusCode`, and the body is in `OriginalMessage`.

### A page that r-1 cannot process

When r-1 cannot process a page, a real-time call returns HTTP 422 with this body:

```json
{"detail": "input could not be processed"}
```

- SageMaker shows it as a 424 `ModelError` with `OriginalStatusCode` 422.
- Do not retry the page. A retry gives the same result.
- In a batch transform job, the record of that page gets status 200, and its `.out` file holds the same body. The job continues.

## Batch transform

Put one JSON request body (either body) in each S3 object. Run a transform job with these settings:

| Setting | Value |
|---|---|
| `ContentType` | `application/json` |
| `SplitType` | `None` |
| `BatchStrategy` | `SingleRecord` |
| `MaxPayloadInMB` | `6` |

Each result is written as `<input>.out`, with the same JSON as the real-time response. Check each `.out` file for `detail` before you read `blocks`. For a chat body, the blocks are in `choices[0].message.content`. A page that r-1 cannot process gives `{"detail": "input could not be processed"}` and does not stop the job.

Any other error on a record fails the whole transform job, and that record gets no `.out` file. Examples are an image that is not a readable PNG or JPEG, a body that is not valid, or a timeout. Check each image before you upload it. If a job fails, read its `FailureReason`, and compare the input objects with the `.out` objects to find the pages without output.

## Billing

Your offer sets the price. A per-request offer bills each real-time call that returns a 2XX response, and does not bill other responses, such as the 422. Batch transform bills per instance hour. AWS bills the endpoint instance while the endpoint exists.

## Limits

- One page per request.
- PNG or JPEG only. Convert other formats, such as PDF or TIFF, to one image for each page before you send them.
- The JSON body, with the base64 image, must be at most 6 MB.
- A decoded image must have at most 25,165,824 pixels. r-1 scales the image down to `max_pixels` (default 3,145,728) before it reads the page.
- The endpoint returns 504 `inference timed out` when a page takes longer than 55 seconds. SageMaker stops a real-time request after 60 seconds.
- The endpoint has no network access.

## Samples

[`samples`](samples) holds three synthetic pages, request bodies, and example answers of model package version 3 for those pages. The content of an answer can differ in a later version.

| Page | Request | Response |
|---|---|---|
| [`invoice.png`](samples/invoice.png) | [`invoice.request.json`](samples/invoice.request.json) (page-parse) | [`invoice.response.json`](samples/invoice.response.json) |
| [`invoice.png`](samples/invoice.png) | [`invoice.chat.request.json`](samples/invoice.chat.request.json) (chat) | [`invoice.chat.response.json`](samples/invoice.chat.response.json) |
| [`text.png`](samples/text.png) | Page-parse body of the image | [`text.response.json`](samples/text.response.json) |
| [`rotated.png`](samples/rotated.png) | Page-parse body of the image | [`rotated.response.json`](samples/rotated.response.json) |

`rotated.png` is a page that is turned by 90 degrees. Its boxes are on the image as you sent it.
