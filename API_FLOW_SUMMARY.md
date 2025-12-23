# API Flow: Instant Response + Get Full Details by ID

## Flow Overview

1. **POST /api/analyze** → Returns instantly with `id` and `event` name
2. **Use the ID** → Get full details via WebSocket event or HTTP endpoint
3. **Receive full data** → Complete result comes via event/endpoint

---

## Step 1: Upload (Returns Instantly)

```dart
POST /api/analyze
Body: { video: File }

Response (202 - Returns in <1 second):
{
  "success": true,
  "id": "abc123",                    // ← Use this ID
  "session_id": "abc123",            // Alias
  "event": "job_complete",           // ← Event name to listen for
  "status": "pending",
  "websocket": {
    "room": "abc123",
    "listen_event": "job_complete"   // ← Listen for this
  },
  "http_endpoint": "/api/status/abc123"  // ← Or use this
}
```

---

## Step 2: Get Full Details Using ID

### Option A: WebSocket (Recommended)

```dart
// 1. Connect
socket.connect();

// 2. Subscribe using the ID
socket.emit('subscribe', {'session_id': 'abc123'});

// 3. Listen for the event name
socket.on('job_complete', (data) {
  // Full result data is here!
  var fullData = data['result'];
  // Contains: total_shots, correct_shots, shot_chunks, etc.
});
```

### Option B: HTTP Endpoint

```dart
// Use the ID with the endpoint
GET /api/status/abc123

Response (when completed):
{
  "session_id": "abc123",
  "status": "completed",
  "progress": 100,
  "result": {
    // Full result data here!
    "total_shots": 15,
    "correct_shots": 12,
    "shot_chunks": [...],
    "ball_hits": [...],
    "processing_time_seconds": 96.5
  }
}
```

---

## Complete Flutter Example

```dart
// Step 1: Upload - get ID and event instantly
var uploadResponse = await service.uploadVideo(videoFile);
String id = uploadResponse['id'];           // "abc123"
String event = uploadResponse['event'];     // "job_complete"
String endpoint = uploadResponse['http_endpoint']; // "/api/status/abc123"

// Step 2: Use ID to get full details

// Method A: WebSocket
service.connectWebSocket();
service.subscribeToSession(id);
service.onJobComplete((data) {
  var fullResult = data['result']; // Complete data!
});

// Method B: HTTP
var status = await http.get('$baseUrl$endpoint');
var fullResult = status['result']; // Complete data!
```

---

## Key Points

✅ **Instant Response**: POST returns in <1 second with `id` and `event`  
✅ **Use ID**: Use the `id` to get full details  
✅ **Event Name**: Listen for the `event` name (usually `job_complete`)  
✅ **Full Data**: Complete result comes via event or HTTP endpoint  
✅ **Both Methods**: WebSocket (real-time) or HTTP (polling) work

---

## Response Structure

### Instant Upload Response
- `id` - Use this to get full details
- `event` - Event name to listen for
- `http_endpoint` - HTTP endpoint to get full details

### Full Details Response (via event or endpoint)
- `result` - Complete analysis data:
  - `total_shots`
  - `correct_shots`
  - `shot_chunks` (array of detailed analysis)
  - `ball_hits`
  - `processing_time_seconds`
  - `mongo_id`

