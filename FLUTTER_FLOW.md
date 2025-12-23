# Flutter Integration Flow

Complete guide for integrating video processing with WebSocket in Flutter.

## Flow Overview

1. **Upload Video** → POST `/api/analyze` → Get `{channel, event, id}`
2. **Connect WebSocket** → Subscribe to `channel`
3. **Wait for Event** → Listen for `event` (`job_complete`)
4. **Receive Data** → Full result data in `data.result`
5. **Display Results** → Show data on UI

---

## Step-by-Step Flutter Implementation

### 1. Dependencies

Add to `pubspec.yaml`:

```yaml
dependencies:
  flutter:
    sdk: flutter
  socket_io_client: ^2.0.3+1
  http: ^1.1.0
  dio: ^5.4.0
```

### 2. Service Class

```dart
import 'dart:async';
import 'dart:io';
import 'package:dio/dio.dart';
import 'package:socket_io_client/socket_io_client.dart' as IO;

class VideoProcessingService {
  final String baseUrl;
  IO.Socket? socket;
  
  VideoProcessingService({required this.baseUrl});
  
  // Upload video - returns instantly with channel, event, id
  Future<Map<String, dynamic>> uploadVideo(File videoFile) async {
    try {
      var dio = Dio();
      var formData = FormData.fromMap({
        'video': await MultipartFile.fromFile(
          videoFile.path,
          filename: videoFile.path.split('/').last,
        ),
      });
      
      var response = await dio.post(
        '$baseUrl/api/analyze',
        data: formData,
        options: Options(
          headers: {'Content-Type': 'multipart/form-data'},
        ),
      );
      
      // Returns 202 Accepted instantly
      if (response.statusCode == 202) {
        return {
          'channel': response.data['channel'],
          'event': response.data['event'],
          'id': response.data['id'],
          'queue_info': response.data['queue_info'],
        };
      } else {
        throw Exception('Upload failed: ${response.statusMessage}');
      }
    } catch (e) {
      throw Exception('Error uploading video: $e');
    }
  }
  
  // Connect to WebSocket
  void connectWebSocket() {
    socket = IO.io(
      baseUrl,
      IO.OptionBuilder()
          .setTransports(['websocket'])
          .enableAutoConnect()
          .build(),
    );
    
    socket!.onConnect((_) {
      print('✅ WebSocket connected');
    });
    
    socket!.onDisconnect((_) {
      print('❌ WebSocket disconnected');
    });
    
    socket!.onError((error) {
      print('⚠️ WebSocket error: $error');
    });
  }
  
  // Subscribe to channel
  void subscribeToChannel(String channel) {
    if (socket == null || !socket!.connected) {
      connectWebSocket();
    }
    
    socket!.emit('subscribe', {'session_id': channel});
    print('📡 Subscribed to channel: $channel');
  }
  
  // Wait for job completion and return full result
  Future<Map<String, dynamic>> waitForCompletion(
    String channel,
    String eventName,
  ) async {
    final completer = Completer<Map<String, dynamic>>();
    
    // Listen for the completion event
    socket!.once(eventName, (data) {
      print('✅ Job complete event received');
      if (!completer.isCompleted) {
        // Full result data is in data['result']
        completer.complete(Map<String, dynamic>.from(data));
      }
    });
    
    // Handle errors
    socket!.once('job_error', (data) {
      if (!completer.isCompleted) {
        completer.completeError(Exception(data['error'] ?? 'Unknown error'));
      }
    });
    
    // Subscribe to channel
    subscribeToChannel(channel);
    
    // Wait for completion
    return completer.future;
  }
  
  // Listen for progress updates (optional)
  void onProgressUpdate(Function(Map<String, dynamic>) callback) {
    socket!.on('progress_update', (data) {
      callback(Map<String, dynamic>.from(data));
    });
  }
  
  // Disconnect
  void disconnect() {
    socket?.disconnect();
    socket?.dispose();
  }
}
```

### 3. Flutter Widget Example

```dart
import 'dart:io';
import 'package:flutter/material.dart';
import 'video_processing_service.dart';

class VideoProcessingScreen extends StatefulWidget {
  final File videoFile;
  
  const VideoProcessingScreen({Key? key, required this.videoFile}) 
      : super(key: key);
  
  @override
  _VideoProcessingScreenState createState() => _VideoProcessingScreenState();
}

class _VideoProcessingScreenState extends State<VideoProcessingScreen> {
  final VideoProcessingService _service = VideoProcessingService(
    baseUrl: 'http://127.0.0.1:8000', // Your backend URL
  );
  
  String? _channel;
  String? _event;
  String? _id;
  bool _isLoading = false;
  String _statusMessage = 'Uploading...';
  int _progress = 0;
  Map<String, dynamic>? _result;
  String? _error;
  
  @override
  void initState() {
    super.initState();
    _startProcessing();
  }
  
  Future<void> _startProcessing() async {
    try {
      setState(() {
        _isLoading = true;
        _statusMessage = 'Uploading video...';
      });
      
      // Step 1: Upload video - get channel, event, id
      var uploadResponse = await _service.uploadVideo(widget.videoFile);
      
      setState(() {
        _channel = uploadResponse['channel'];
        _event = uploadResponse['event'];
        _id = uploadResponse['id'];
        _statusMessage = 'Video uploaded. Processing...';
      });
      
      // Step 2: Connect WebSocket
      _service.connectWebSocket();
      
      // Step 3: Listen for progress updates (optional)
      _service.onProgressUpdate((data) {
        setState(() {
          _progress = data['progress'] ?? 0;
          _statusMessage = data['message'] ?? 'Processing...';
        });
      });
      
      // Step 4: Wait for completion event
      if (_channel != null && _event != null) {
        var completionData = await _service.waitForCompletion(_channel!, _event!);
        
        // Full result data is in completionData['result']
        setState(() {
          _isLoading = false;
          _result = completionData['result'];
          _statusMessage = 'Processing complete!';
        });
      }
    } catch (e) {
      setState(() {
        _isLoading = false;
        _error = e.toString();
        _statusMessage = 'Error: $e';
      });
    }
  }
  
  @override
  void dispose() {
    _service.disconnect();
    super.dispose();
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Processing Video')),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          children: [
            // Status info
            if (_channel != null) ...[
              Card(
                child: Padding(
                  padding: const EdgeInsets.all(16.0),
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      Text('Channel: $_channel', style: const TextStyle(fontWeight: FontWeight.bold)),
                      Text('Event: $_event'),
                      Text('ID: $_id'),
                    ],
                  ),
                ),
              ),
              const SizedBox(height: 16),
            ],
            
            // Loading indicator
            if (_isLoading) ...[
              CircularProgressIndicator(value: _progress > 0 ? _progress / 100 : null),
              const SizedBox(height: 16),
              Text(_statusMessage),
              if (_progress > 0) Text('Progress: $_progress%'),
            ],
            
            // Error display
            if (_error != null)
              Card(
                color: Colors.red[50],
                child: Padding(
                  padding: const EdgeInsets.all(16.0),
                  child: Text('Error: $_error', style: const TextStyle(color: Colors.red)),
                ),
              ),
            
            // Results display
            if (_result != null) ...[
              const SizedBox(height: 24),
              const Text(
                'Results:',
                style: TextStyle(fontSize: 20, fontWeight: FontWeight.bold),
              ),
              const SizedBox(height: 16),
              _buildResultsCard(_result!),
            ],
          ],
        ),
      ),
    );
  }
  
  Widget _buildResultsCard(Map<String, dynamic> result) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            _buildResultRow('Total Shots', result['total_shots']?.toString() ?? '0'),
            _buildResultRow('Correct Shots', result['correct_shots']?.toString() ?? '0'),
            _buildResultRow('Incorrect Shots', result['incorrect_shots']?.toString() ?? '0'),
            _buildResultRow(
              'Processing Time',
              '${result['processing_time_seconds']?.toStringAsFixed(2) ?? '0'}s',
            ),
            if (result['mongo_id'] != null)
              _buildResultRow('MongoDB ID', result['mongo_id']),
            
            // Show shot chunks count
            if (result['shot_chunks'] != null)
              _buildResultRow('Shot Chunks', result['shot_chunks'].length.toString()),
          ],
        ),
      ),
    );
  }
  
  Widget _buildResultRow(String label, String value) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 8.0),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceBetween,
        children: [
          Text(label, style: const TextStyle(fontWeight: FontWeight.w500)),
          Text(value, style: const TextStyle(fontSize: 16)),
        ],
      ),
    );
  }
}
```

---

## Complete Flow Diagram

```
┌─────────────┐
│  Upload     │
│  Video      │
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│ POST /api/analyze│
│ Returns:         │
│ {channel, event,│
│  id}            │
└──────┬──────────┘
       │
       ▼
┌──────────────────┐
│ Connect WebSocket│
│ Subscribe to     │
│ channel          │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│ Show Loading UI  │
│ (Progress updates│
│  optional)       │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│ Wait for Event   │
│ "job_complete"   │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│ Receive Data     │
│ data.result      │
│ {total_shots,    │
│  correct_shots,  │
│  shot_chunks...} │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│ Display Results  │
│ on UI            │
└──────────────────┘
```

---

## API Response Structure

### Upload Response (202 Accepted)
```json
{
  "channel": "70978b51-2786-4504-94f8-d92671596878",
  "event": "job_complete",
  "id": "70978b51-2786-4504-94f8-d92671596878",
  "queue_info": {
    "active_jobs": 0,
    "max_concurrent": 3,
    "position": null,
    "status": "processing"
  }
}
```

### WebSocket Event: `job_complete`
```json
{
  "session_id": "70978b51-2786-4504-94f8-d92671596878",
  "status": "completed",
  "progress": 100,
  "message": "Processing complete",
  "result": {
    "success": true,
    "session_id": "70978b51-2786-4504-94f8-d92671596878",
    "total_shots": 15,
    "correct_shots": 12,
    "incorrect_shots": 3,
    "ball_hits": [...],
    "shot_chunks": [...],
    "processing_time_seconds": 96.5,
    "mongo_id": "..."
  }
}
```

---

## Key Points

✅ **Instant Response**: API returns in ~2 seconds with `{channel, event, id}`  
✅ **WebSocket Connection**: Connect and subscribe to `channel`  
✅ **Event Listening**: Listen for `event` name (`job_complete`)  
✅ **Full Data**: Complete result is in `data.result`  
✅ **Display Results**: Show data on UI when event received

---

## Testing

1. Start your backend server
2. Update `baseUrl` in Flutter service
3. Upload a video file
4. Watch the flow:
   - Upload → Get channel/event/id
   - Connect WebSocket
   - Subscribe to channel
   - Wait for `job_complete` event
   - Display results from `data.result`

