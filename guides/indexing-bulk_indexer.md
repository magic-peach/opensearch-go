# Bulk Indexer

> **Runnable example:** [`ExampleNewBulkIndexer`](../opensearchutil/bulk_indexer_example_test.go)

The `opensearchutil.BulkIndexer` batches documents you add one at a time into bulk requests behind the scenes, so you get the throughput of the [Bulk API](indexing-bulk.md) without having to build and size the requests yourself. It flushes automatically once a buffer fills up or a timer elapses, and reports the result of each item through a callback.

## Setup

```go
client, err := opensearchapi.NewClient(opensearchapi.Config{
	Client: opensearch.Config{
		Addresses: []string{"https://localhost:9200"},
	},
})
if err != nil {
	log.Fatalf("Error creating the client: %s", err)
}
defer func() { _ = client.Close() }()

indexer, err := opensearchutil.NewBulkIndexer(opensearchutil.BulkIndexerConfig{
	Client:     client,     // The OpenSearch client
	Index:      "my-index", // The default index name
	NumWorkers: 4,          // The number of worker goroutines (default: number of CPUs)
	FlushBytes: 5e+6,       // The flush threshold in bytes (default: 5MB)
})
if err != nil {
	log.Panicf("Error creating the indexer: %s", err)
}
```

`BulkIndexerConfig` also accepts most of the `Bulk` API's own parameters directly (`Pipeline`, `Refresh`, `Routing`, `Source`, `WaitForActiveShards`, `Timeout`, and so on), applied to every request the indexer sends.

## Adding items

```go
err = indexer.Add(context.Background(), opensearchutil.BulkIndexerItem{
	Action:     "index", // index, create, delete, or update
	DocumentID: "doc-1", // optional
	Body:       strings.NewReader(`{"title":"Test"}`),

	OnSuccess: func(ctx context.Context, item opensearchutil.BulkIndexerItem, res opensearchapi.BulkRespItem) {
		fmt.Printf("indexed %s: %s\n", item.DocumentID, res.Result)
	},
	OnFailure: func(ctx context.Context, item opensearchutil.BulkIndexerItem, res opensearchapi.BulkRespItem, err error) {
		if err != nil {
			log.Printf("error: %s", err)
		} else {
			log.Printf("error: %s: %s", res.Error.Type, res.Error.Reason)
		}
	},
})
```

`Add` queues the item and returns once it is buffered, not once it is sent; `OnSuccess` and `OnFailure` report the outcome of that specific item when its bulk request eventually completes.

Items that carry a `DocumentID` are routed to a fixed worker based on a hash of the ID, so a create followed by an update for the same document is always sent in the order you added them. Items without a `DocumentID` are spread across workers round robin.

## Flushing and closing

The indexer flushes a worker's buffer automatically once it reaches `FlushBytes` or `FlushInterval` elapses, whichever comes first. Call `Flush` to force every item queued so far out immediately without stopping the indexer:

```go
if err := indexer.Flush(context.Background()); err != nil {
	log.Printf("flush error: %s", err)
}
```

A non-nil error from `Flush` (or `Close`) means the request itself failed to go through; documents the cluster rejected individually still go to their own `OnFailure` callback rather than being reported here.

Once you're done adding items, call `Close` to drain everything that's left and stop the workers:

```go
if err := indexer.Close(context.Background()); err != nil {
	log.Panicf("Unexpected error: %s", err)
}
```

`Add` and `Flush` both panic if called after `Close`.

## Statistics

`Stats` returns a point-in-time snapshot of what the indexer has done so far:

```go
stats := indexer.Stats()
if stats.NumFailed > 0 {
	log.Printf("indexed %d documents with %d errors", stats.NumFlushed, stats.NumFailed)
}
```

`NumFlushed` counts successful items broken down further into `NumIndexed`, `NumCreated`, `NumUpdated`, and `NumDeleted`; `NumRequests` counts the underlying bulk requests sent.
