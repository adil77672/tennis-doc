# API Flow: Upload → Get ID → Retrieve Data

## Step 1: Upload Video

```bash
curl --location 'http://127.0.0.1:8000/api/analyze' \
--form 'video=@"/Users/macbookpro/Downloads/mixkit-tennis-players-at-an-outdoor-court-869-hd-ready.mp4"'
```

**Response (202 Accepted - Returns instantly):**
```json
{
  "channel": "abc123-def456-ghi789",
  "event": "job_complete",
  "id": "abc123-def456-ghi789"
}
```

## Step 2: Wait for Job Complete

The job processes in the background. When complete:
- Data is **automatically saved to MongoDB**
- WebSocket event `job_complete` is emitted (if subscribed)
- You can retrieve data using the `id`

## Step 3: Get Data by ID

```bash
curl --location 'http://127.0.0.1:8000/api/data/abc123-def456-ghi789'
```

**Response (200 OK):**
```json
{
  "id": "abc123-def456-ghi789",
  "session_id": "abc123-def456-ghi789",
  "status": "completed",
  "created_at": "2025-01-15T10:30:00",
  "data": {
    "total_shots": 15,
    "correct_shots": 12,
    "incorrect_shots": 3,
    "shot_chunks": [...],
    "ball_hits": [...],
    "processing_time_seconds": 96.5,
    "analysis": {...},
    "mongo_id": "507f1f77bcf86cd799439011"
  }
}
```

## Alternative: Check Status

```bash
curl --location 'http://127.0.0.1:8000/api/status/abc123-def456-ghi789'
```

**Response (while processing):**
```json
{
  "session_id": "abc123-def456-ghi789",
  "status": "processing",
  "progress": 45,
  "message": "Processing video..."
}
```

**Response (when completed):**
```json
{
  "session_id": "abc123-def456-ghi789",
  "status": "completed",
  "progress": 100,
  "message": "Processing complete",
  "result": {
    "total_shots": 15,
    "correct_shots": 12,
    ...
  }
}
```

## WebSocket Flow (Optional)

1. **Connect to WebSocket:**
   ```javascript
   const socket = io('http://127.0.0.1:8000');
   ```

2. **Subscribe to channel (use `channel` from upload response):**
   ```javascript
   socket.emit('subscribe', { session_id: 'abc123-def456-ghi789' });
   ```

3. **Listen for event (use `event` from upload response):**
   ```javascript
   socket.on('job_complete', (data) => {
     console.log('Complete!', data.result);
     // Full data is in data.result
   });
   ```

## Summary

1. **Upload** → Get `{channel, event, id}`
2. **Wait** → Job processes in background
3. **Get Data** → Use `/api/data/{id}` to retrieve from DB
4. **Data Saved** → Automatically saved to MongoDB on completion

## Endpoints

- `POST /api/analyze` - Upload video, get ID
- `GET /api/data/{id}` - Get full data by ID from DB
- `GET /api/status/{id}` - Get job status
- WebSocket: Subscribe to `channel`, listen for `event`

