# Postable

<img alt="Postable Task Service" src="https://github.com/aol/postable/raw/master/docs/img/postable.png" height="200">

A simple task distribution and result collection service.

  [![NPM Version][npm-image]][npm-url]
  [![NPM License][npm-license-image]][npm-url]
  [![Build][travis-image]][travis-url]
  [![Test Coverage][coveralls-image]][coveralls-url]

Postable can receive tasks and distribute them to listeners.
Results from listeners are then forwarded back to the original caller as line-delimited JSON.

Postable can be clustered, but instances must share the same Redis master.

## Workflow

1. Listeners connect to Postable specifying the *buckets* to listen to.
2. Tasks are submitted to Postable under a specific bucket.
3. Listeners on that bucket receive the task as a line-delimited JSON entry.
4. Listeners respond to Postable with the result for that specific task.
5. Postable streams all results for the task back to the caller as line-delimited JSON.

## Installation

### Server Installation

- Install Redis
- Install Postable `npm install postable`
- Set configuration as environment variables
- Run service `bin/postable`

### Listener Installation

- Install Postable `npm install postable`
- Set configuration as environment variables
- Run client `bin/postable-listener`

## Configuration

### Environment Variables (Server)

- `POSTABLE_PORT`  
  *Optional* (defaults to `3000`). The port to listen on.

- `POSTABLE_BROADCAST`  
  *Optional* (defaults to none).
  A semicolon-delimited set of base URIs of other postable services for task broadcasting (see **Task Broadcasting** below).    
  Example: `http://foo.example.com;http://bar.example.com;http://baz.example.com`

- `POSTABLE_REDIS_HOST`  
  *Optional* (defaults to `127.0.0.1`). The redis host to use.

- `POSTABLE_REDIS_PORT`  
  *Optional* (defaults to `6379`). The redis port to connect to.

- `POSTABLE_REDIS_PASS`  
  *Optional* (defaults to none). The auth password for redis.

- `POSTABLE_REDIS_PREFIX`  
  *Optional* (defaults to `postable_`). The prefix for redis keys and channels.

- `POSTABLE_HEARTBEAT_MILLIS`  
  *Optional* (defaults to 5 seconds, `5000`). How often to send a heartbeat message so client connections don't close.

- `POSTABLE_LISTENER_TIMEOUT_SECONDS`  
  *Optional* (defaults to 20 seconds, `20`). How long to keep listener data in redis without an active client connection.

- `POSTABLE_LISTENER_SET_TIMEOUT_SECONDS`  
  *Optional* (defaults to 30 mins, `1800`). How long to keep listener set data in redis.

- `POSTABLE_LAST_TASK_TIMEOUT_SECONDS`  
  *Optional* (defaults to 7 days, `604800`). How long to keep the last task per bucket.

### Environment Variables (Server and Listener)

- `POSTABLE_LOG_FILE`  
  *Optional* (defaults to console). Where to log data.

- `POSTABLE_LOG_LEVEL`  
  *Optional* (defaults to `info`). The minimum level to log.

- `POSTABLE_AUTH_USER`  
  *Optional* (defaults to none). A username for basic HTTP authentication to the service.

- `POSTABLE_AUTH_PASS`  
  *Optional* (defaults to none). A password for basic HTTP authentication to the service.

### Environment Variables (Listener)

- `POSTABLE_LISTEN_BASE_URL`  
  **Required**. The base URL for the Postable service.

- `POSTABLE_LISTEN_BUCKETS_HTTP_URL`  
  **Required**. A URL to get the listener buckets from. The buckets will be refreshed periodically.

- `POSTABLE_LISTEN_FORWARD_HTTP_URL`  
  **Required**. A URL to forward the task to. Will forward as a JSON body:
  ```js
  {
    "task":  { /* task data */ }
  }
  ```

- `POSTABLE_MAX_CONCURRENT_FORWARDS`  
  *Optional* (defaults to `50`). The max number of concurrent task forwards to send to the forward URL.

- `POSTABLE_LISTEN_RECONNECT_RATE`  
  *Optional* (defaults to 5 seconds, `5`). How soon to reconnect to postable in the case of an error.

- `POSTABLE_LISTEN_BUCKETS_REFRESH_RATE`  
  *Optional* (defaults to 1 minute, `60`). How often to refresh buckets.
  
- `POSTABLE_LISTEN_BUCKETS_RECONNECT_RATE`  
  *Optional* (defaults to 5 seconds, `5`). How soon to reattempt to get buckets in the case of an error.
  
- `POSTABLE_LISTEN_FORWARD_ATTEMPTS`  
  *Optional* (defaults to `2`). How many times to attempt forwarding the data to the HTTP URL.

- `POSTABLE_LISTEN_LISTENER_DATA`  
  *Optional* (defaults to an empty object). Listener data (as JSON) to send to Postable.

## Usage

### Listening for Tasks

```js
POST /listeners/
{
  "buckets": [ "bucket-1", "bucket-2", ... ],
  
  ... additional listener data ...
}
=> 200
{ "id": <taskId>, "started": <dateString>, "listenerId": <listenerId>, "data": <taskData> }
...
```

To listen for tasks on buckets, a client will `POST /listeners/` with a body containing the `buckets` to listen to (as an array of strings).

This will be a **long-poll** request and as tasks come in they will be *streamed* to the client as *line-delimited JSON*. 
The connection **is never closed** by the service.

Each task will contain an `id`, `started`, `listenerId`, and the `data` from the task.

### Sending a Task

```js
POST /buckets/<bucket>/tasks/
{
  ... task data ...
}
=> 200
{ "meta": { "listenersPending": [ ... ] } }
{ "listener": { "buckets": [...], "id": <listenerId>, "started": <dateString>, ... }, "timeout": false, "data": <result> }
...
```

To send a task to a bucket, simply `POST /buckets/<bucket>/tasks/` with the task data as a JSON object.

The response will be a stream of *line-delimited JSON*. The first entry will be a meta entry containing `listenersPending` (an array of listener IDs).

This task will be given a unique task ID and sent to all listeners. As listeners respond to the task with results, those results
will be *streamed* back to this response. Each entry will contain the listener ID sending the result.
Once all results have been received, the connection will close. 

#### Timeout

If the timeout is reached the connection will close with additional entries for each timed out listener with a property `timeout` set to `true`.
This timeout can be configured using `?timeout=<seconds>`.

#### Task Send Options

|Option||
|:---|:---|
|`timeout`|Query-string value. Defaults to `30`. Specify the listener timeout (in seconds)|
|`max`|Query-string value. Defaults to no maximum. Specify the maximum number of listeners to receive the task.|

### Task Broadcasting

A task can be broadcast across data-centers using by calling the broadcast endpoint. 
Postable must be configured to use this feature (see the `POSTABLE_BROADCAST` option under **Configuration**).

This endpoint is similar to **Sending a Task**, but begins with `/broadcast/`:

```js
POST /broadcast/buckets/<bucket>/tasks/
{
  ... task data ...
}
=> 200
{ "clusterId": "a...", "meta": { "listenersPending": [ ... ] } }
{ "clusterId": "a...", "listener": { "buckets": [...], "id": <listenerId>, "started": <dateString>, ... }, "timeout": false, "data": <result> }
{ "clusterId": "b...", "meta": { "listenersPending": [ ... ] } }
...
```

The response will be a stream of *line-delimited JSON*. Results from various clusters will be streamed back.
One meta item containing `listenersPending` will be sent from *each cluster*.

#### Task Broadcasting Options

|Option||
|:---|:---|
|`timeout`|Same as the above *Task Send* option; forwarded to each cluster.|
|`max`|Same as the above *Task Send* option; forwarded to each cluster. Specifies the maximum number of listeners (per cluster) to receive the task. That means each cluster will send the task to the number of listeners specified by `?max`.|
|`cluster`|Defaults to nothing. Only clusters with the ID specified will respond. This can also be an array (`?cluster[0]=<id>&cluster[1]=<id>&...`).|

### Responding to a Task

```js
POST /tasks/<taskId>/results/<listenerId>
{
  ... task result ...
}
=> 200
```

To respond to a task from a listener, simply `POST /tasks/<taskId>/results/<listenerId>` with the task result as a JSON object.
 
The `<taskId>` and `<listenerId>` should come from the initial task sent (see **Listening for Tasks**).

### Getting the Last Task

```js
GET /buckets/<bucket>/tasks/last
=> 200
{
  ... task data ...
}
```

Return the last task submitted to the given bucket as JSON, or `null` if there was none.

### Getting Bucket Listeners

```js
GET /buckets/<bucket>/listeners/
=> 200
{
  [
    { ... listener 1 info ... },
    { ... listener 2 info ... },
    ...
  ]
}
```

Return the last task submitted to the given bucket as JSON, or `null` if there was none.

## Implementation

Postable works by using redis for pub/sub. See below for a high-level sequence diagram:

In the diagram below, `Potable 1` and `Postable 2` can either be the same or different boxes in the same cluster.

<img alt="Sequence Diagram" src="https://github.com/aol/postable/raw/master/docs/img/sequence.png" width="650">

## License

  [MIT](LICENSE)

[npm-image]: https://img.shields.io/npm/v/postable.svg
[npm-license-image]: https://img.shields.io/npm/l/postable.svg
[npm-url]: https://npmjs.org/package/postable
[travis-image]: https://img.shields.io/travis/aol/postable/master.svg
[travis-url]: https://travis-ci.org/aol/postable
[coveralls-image]: https://img.shields.io/coveralls/aol/postable/master.svg
[coveralls-url]: https://coveralls.io/github/aol/postable
