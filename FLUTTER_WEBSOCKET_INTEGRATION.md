# Flutter Integration Guide

This guide shows how to integrate video processing in your Flutter app using **both WebSocket (real-time)** and **HTTP endpoints (polling)** methods.

## Dependencies

Add to your `pubspec.yaml`:

```yaml
dependencies:
  flutter:
    sdk: flutter
  socket_io_client: ^2.0.3+1  # For WebSocket (optional)
  http: ^1.1.0  # For HTTP endpoints
  dio: ^5.4.0  # For file uploads
```

**Note**: You can use either WebSocket OR HTTP endpoints. Both methods work independently.

## Quick Start Flow

1. **Upload Video** → Returns instantly with `id` and `event` name
2. **Use the ID** → Get full details via WebSocket event or HTTP endpoint
3. **Receive Data** → Full result data comes via event or endpoint

### Example Response (Returns Instantly)
```json
{
  "success": true,
  "id": "abc123",              // ← Use this ID to get full details
  "event": "job_complete",      // ← Listen for this event
  "http_endpoint": "/api/status/abc123"  // ← Or use this endpoint
}
```

### Then Get Full Details Using ID:

**WebSocket:**
```dart
// Use the ID to subscribe
socket.emit('subscribe', {'session_id': id});
// Listen for the event name
socket.on(event, (data) {
  var fullData = data['result']; // Complete result here
});
```

**HTTP:**
```dart
// Use the ID with endpoint
var response = await http.get('$baseUrl$http_endpoint');
var fullData = response['result']; // Complete result here
```

---

## Two Integration Methods

### Method 1: WebSocket (Real-time Updates) ⚡
- Instant progress updates
- No polling needed
- Lower server load
- **Recommended for production**

### Method 2: HTTP Endpoints (Polling) 🔄
- Simple to implement
- Works without WebSocket support
- Good fallback option
- Poll every 5-10 seconds

---

## Complete Flutter Examples

### Method 1: WebSocket Service

```dart
import 'dart:async';
import 'dart:io';
import 'package:dio/dio.dart';
import 'package:http/http.dart' as http;
import 'package:socket_io_client/socket_io_client.dart' as IO;

class VideoProcessingService {
  final String baseUrl;
  IO.Socket? socket;
  
  VideoProcessingService({required this.baseUrl});
  
  // Connect to WebSocket server
  void connectWebSocket() {
    socket = IO.io(
      baseUrl,
      IO.OptionBuilder()
          .setTransports(['websocket'])
          .enableAutoConnect()
          .build(),
    );
    
    socket!.onConnect((_) {
      print('WebSocket connected');
    });
    
    socket!.onDisconnect((_) {
      print('WebSocket disconnected');
    });
    
    socket!.onError((error) {
      print('WebSocket error: $error');
    });
  }
  
  // Upload video - returns instantly with ID and event name
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
      
      // Returns 202 Accepted instantly with ID and event name
      if (response.statusCode == 202) {
        return {
          'id': response.data['id'] ?? response.data['session_id'],
          'session_id': response.data['session_id'],
          'event': response.data['event'] ?? 'job_complete',
          'websocket': response.data['websocket'],
          'http_endpoint': response.data['http_endpoint'],
        };
      } else {
        throw Exception('Upload failed: ${response.statusMessage}');
      }
    } catch (e) {
      throw Exception('Error uploading video: $e');
    }
  }
  
  // Get session_id from upload response (for backward compatibility)
  Future<String> uploadVideoGetId(File videoFile) async {
    var response = await uploadVideo(videoFile);
    return response['id'] as String;
  }
  
  // Wait for job completion event and return full result data
  // Uses the ID from upload response and listens for the event name
  Future<Map<String, dynamic>> waitForCompletion(
    String sessionId, {
    String eventName = 'job_complete',
  }) async {
    final completer = Completer<Map<String, dynamic>>();
    
    // Set up one-time listener for completion event
    socket!.once(eventName, (data) {
      if (!completer.isCompleted) {
        // Full result data is in data['result']
        completer.complete(Map<String, dynamic>.from(data));
      }
    });
    
    // Also handle errors
    socket!.once('job_error', (data) {
      if (!completer.isCompleted) {
        completer.completeError(Exception(data['error'] ?? 'Unknown error'));
      }
    });
    
    // Subscribe to session using the ID
    subscribeToSession(sessionId);
    
    // Wait for completion event - returns full details
    return completer.future;
  }
  
  // Get full details using ID (from upload response)
  Future<Map<String, dynamic>> getFullDetails(String id, String eventName) async {
    return waitForCompletion(id, eventName: eventName);
  }
  
  // Subscribe to session updates
  void subscribeToSession(String sessionId) {
    if (socket == null || !socket!.connected) {
      connectWebSocket();
    }
    
    socket!.emit('subscribe', {'session_id': sessionId});
  }
  
  // Listen for progress updates
  void onProgressUpdate(Function(Map<String, dynamic>) callback) {
    socket!.on('progress_update', (data) {
      callback(Map<String, dynamic>.from(data));
    });
  }
  
  // Listen for completion
  void onJobComplete(Function(Map<String, dynamic>) callback) {
    socket!.on('job_complete', (data) {
      callback(Map<String, dynamic>.from(data));
    });
  }
  
  // Listen for errors
  void onJobError(Function(Map<String, dynamic>) callback) {
    socket!.on('job_error', (data) {
      callback(Map<String, dynamic>.from(data));
    });
  }
  
  // Unsubscribe from session
  void unsubscribeFromSession(String sessionId) {
    socket!.emit('unsubscribe', {'session_id': sessionId});
  }
  
  // Disconnect WebSocket
  void disconnect() {
    socket?.disconnect();
    socket?.dispose();
  }
}
```

### Method 2: HTTP Endpoint Service (No WebSocket)

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:io';
import 'package:dio/dio.dart';
import 'package:http/http.dart' as http;

class VideoProcessingServiceHTTP {
  final String baseUrl;
  Timer? _pollTimer;
  
  VideoProcessingServiceHTTP({required this.baseUrl});
  
  // Upload video - returns instantly with ID and event name
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
      
      // Returns 202 Accepted instantly with ID and event
      if (response.statusCode == 202) {
        return {
          'id': response.data['id'] ?? response.data['session_id'],
          'session_id': response.data['session_id'],
          'event': response.data['event'] ?? 'job_complete',
          'http_endpoint': response.data['http_endpoint'],
        };
      } else {
        throw Exception('Upload failed: ${response.statusMessage}');
      }
    } catch (e) {
      throw Exception('Error uploading video: $e');
    }
  }
  
  // Get ID from upload response (for backward compatibility)
  Future<String> uploadVideoGetId(File videoFile) async {
    var response = await uploadVideo(videoFile);
    return response['id'] as String;
  }
  
  // Get job status from HTTP endpoint
  Future<Map<String, dynamic>> getStatus(String sessionId) async {
    final response = await http.get(
      Uri.parse('$baseUrl/api/status/$sessionId'),
    );
    
    if (response.statusCode == 200) {
      return json.decode(response.body) as Map<String, dynamic>;
    } else if (response.statusCode == 404) {
      throw Exception('Job not found');
    } else {
      throw Exception('Failed to get status: ${response.statusCode}');
    }
  }
  
  // Get full details using ID from upload response
  // Use the ID with HTTP endpoint to get full result data
  Future<Map<String, dynamic>> getFullDetails(String id) async {
    return await getStatus(id);
  }
  
  // Poll status until completion using ID
  Future<Map<String, dynamic>> waitForCompletion(
    String id, {
    Duration pollInterval = const Duration(seconds: 5),
    Function(Map<String, dynamic>)? onProgress,
  }) async {
    final completer = Completer<Map<String, dynamic>>();
    
    _pollTimer = Timer.periodic(pollInterval, (timer) async {
      try {
        var status = await getStatus(sessionId);
        
        // Call progress callback if provided
        if (onProgress != null) {
          onProgress(status);
        }
        
        // Check if completed or failed
        if (status['status'] == 'completed') {
          timer.cancel();
          if (!completer.isCompleted) {
            completer.complete(status);
          }
        } else if (status['status'] == 'failed') {
          timer.cancel();
          if (!completer.isCompleted) {
            completer.completeError(Exception(status['error'] ?? 'Processing failed'));
          }
        }
      } catch (e) {
        timer.cancel();
        if (!completer.isCompleted) {
          completer.completeError(e);
        }
      }
    });
    
    return completer.future;
  }
  
  // Get queue status
  Future<Map<String, dynamic>> getQueueStatus() async {
    final response = await http.get(
      Uri.parse('$baseUrl/api/queue'),
    );
    
    if (response.statusCode == 200) {
      return json.decode(response.body) as Map<String, dynamic>;
    } else {
      throw Exception('Failed to get queue status');
    }
  }
  
  // Cancel polling
  void cancelPolling() {
    _pollTimer?.cancel();
    _pollTimer = null;
  }
}
```

### Method 1: WebSocket Widget

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
    baseUrl: 'https://your-render-app.onrender.com', // Your backend URL
  );
  
  String? _sessionId;
  String _status = 'Uploading...';
  int _progress = 0;
  String _message = '';
  Map<String, dynamic>? _result;
  String? _error;
  
  @override
  void initState() {
    super.initState();
    _initializeProcessing();
  }
  
  Future<void> _initializeProcessing() async {
    try {
      // Step 1: Upload video - returns instantly with ID and event name
      var uploadResponse = await _service.uploadVideo(widget.videoFile);
      _sessionId = uploadResponse['id'] as String;
      String eventName = uploadResponse['event'] as String; // 'job_complete'
      
      setState(() {
        _status = 'processing';
        _message = 'Video uploaded, waiting for processing...';
      });
      
      // Step 2: Connect to WebSocket
      _service.connectWebSocket();
      
      // Step 3: Set up progress listeners (optional)
      _service.onProgressUpdate((data) {
        setState(() {
          _progress = data['progress'] ?? 0;
          _message = data['message'] ?? '';
          _status = data['status'] ?? 'processing';
        });
      });
      
      // Step 4: Use the ID to get full details via event
      if (_sessionId != null) {
        _service.subscribeToSession(_sessionId!);
        
        // Listen for the event name from upload response
        // This event will contain full result data
        _service.onJobComplete((data) {
          setState(() {
            _status = 'completed';
            _progress = 100;
            _result = data['result']; // Full result data is here!
            _message = 'Processing complete!';
          });
        });
        
        _service.onJobError((data) {
          setState(() {
            _status = 'failed';
            _error = data['error'] ?? 'Unknown error';
            _message = 'Processing failed';
          });
        });
      }
    } catch (e) {
      setState(() {
        _status = 'failed';
        _error = e.toString();
      });
    }
  }
  
  // Alternative: Use async/await - Get ID from upload, then get full details
  Future<void> _initializeProcessingAsync() async {
    try {
      // Step 1: Upload video - returns instantly with ID and event
      var uploadResponse = await _service.uploadVideo(widget.videoFile);
      _sessionId = uploadResponse['id'] as String;
      String eventName = uploadResponse['event'] as String;
      
      setState(() {
        _status = 'processing';
        _message = 'Video uploaded, processing...';
      });
      
      // Step 2: Connect and use ID to get full details
      _service.connectWebSocket();
      
      if (_sessionId != null) {
        // Use the ID to wait for the event - returns full result data
        var completionData = await _service.getFullDetails(_sessionId!, eventName);
        
        setState(() {
          _status = 'completed';
          _progress = 100;
          _result = completionData['result']; // Full result data
          _message = 'Processing complete!';
        });
      }
    } catch (e) {
      setState(() {
        _status = 'failed';
        _error = e.toString();
      });
    }
  }
  
  @override
  void dispose() {
    if (_sessionId != null) {
      _service.unsubscribeFromSession(_sessionId!);
    }
    _service.disconnect();
    super.dispose();
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Processing Video'),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            // Progress indicator
            LinearProgressIndicator(
              value: _progress / 100,
              backgroundColor: Colors.grey[300],
              valueColor: AlwaysStoppedAnimation<Color>(
                _status == 'failed' ? Colors.red : Colors.blue,
              ),
            ),
            const SizedBox(height: 16),
            
            // Status text
            Text(
              'Status: $_status',
              style: const TextStyle(
                fontSize: 18,
                fontWeight: FontWeight.bold,
              ),
            ),
            const SizedBox(height: 8),
            
            // Progress percentage
            Text(
              'Progress: $_progress%',
              style: const TextStyle(fontSize: 16),
            ),
            const SizedBox(height: 8),
            
            // Message
            Text(
              _message,
              style: TextStyle(
                fontSize: 14,
                color: Colors.grey[600],
              ),
            ),
            const SizedBox(height: 24),
            
            // Error display
            if (_error != null)
              Container(
                padding: const EdgeInsets.all(12),
                decoration: BoxDecoration(
                  color: Colors.red[50],
                  borderRadius: BorderRadius.circular(8),
                  border: Border.all(color: Colors.red),
                ),
                child: Text(
                  'Error: $_error',
                  style: const TextStyle(color: Colors.red),
                ),
              ),
            
            // Results display
            if (_result != null) ...[
              const SizedBox(height: 24),
              const Text(
                'Results:',
                style: TextStyle(
                  fontSize: 18,
                  fontWeight: FontWeight.bold,
                ),
              ),
              const SizedBox(height: 8),
              _buildResultsCard(_result!),
            ],
            
            const Spacer(),
            
            // Action buttons
            if (_status == 'completed' || _status == 'failed')
              ElevatedButton(
                onPressed: () => Navigator.pop(context),
                child: const Text('Done'),
              ),
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
          ],
        ),
      ),
    );
  }
  
  Widget _buildResultRow(String label, String value) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4.0),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceBetween,
        children: [
          Text(
            label,
            style: const TextStyle(fontWeight: FontWeight.w500),
          ),
          Text(value),
        ],
      ),
    );
  }
}
```

### Method 2: HTTP Endpoint Widget

```dart
import 'dart:io';
import 'package:flutter/material.dart';
import 'video_processing_service_http.dart';

class VideoProcessingScreenHTTP extends StatefulWidget {
  final File videoFile;
  
  const VideoProcessingScreenHTTP({Key? key, required this.videoFile}) 
      : super(key: key);
  
  @override
  _VideoProcessingScreenHTTPState createState() => _VideoProcessingScreenHTTPState();
}

class _VideoProcessingScreenHTTPState extends State<VideoProcessingScreenHTTP> {
  final VideoProcessingServiceHTTP _service = VideoProcessingServiceHTTP(
    baseUrl: 'https://your-render-app.onrender.com',
  );
  
  String? _sessionId;
  String _status = 'Uploading...';
  int _progress = 0;
  String _message = '';
  Map<String, dynamic>? _result;
  String? _error;
  
  @override
  void initState() {
    super.initState();
    _initializeProcessing();
  }
  
  Future<void> _initializeProcessing() async {
    try {
      // Step 1: Upload video - returns instantly with ID and event
      var uploadResponse = await _service.uploadVideo(widget.videoFile);
      _sessionId = uploadResponse['id'] as String;
      String httpEndpoint = uploadResponse['http_endpoint'] as String;
      
      setState(() {
        _status = 'processing';
        _message = 'Video uploaded, processing...';
      });
      
      // Step 2: Use the ID to poll status until completion
      if (_sessionId != null) {
        var finalStatus = await _service.waitForCompletion(
          _sessionId!,  // Use ID from upload response
          onProgress: (status) {
            // Update UI with progress
            setState(() {
              _progress = status['progress'] ?? 0;
              _message = status['message'] ?? '';
              _status = status['status'] ?? 'processing';
            });
          },
        );
        
        // Processing complete - full result data is here
        setState(() {
          _status = 'completed';
          _progress = 100;
          _result = finalStatus['result']; // Full result data
          _message = 'Processing complete!';
        });
      }
    } catch (e) {
      setState(() {
        _status = 'failed';
        _error = e.toString();
      });
    }
  }
  
  @override
  void dispose() {
    _service.cancelPolling();
    super.dispose();
  }
  
  @override
  Widget build(BuildContext context) {
    // Same UI as WebSocket version
    return Scaffold(
      appBar: AppBar(title: const Text('Processing Video')),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          children: [
            LinearProgressIndicator(value: _progress / 100),
            const SizedBox(height: 16),
            Text('Status: $_status', style: const TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
            Text('Progress: $_progress%'),
            Text(_message),
            if (_error != null)
              Text('Error: $_error', style: const TextStyle(color: Colors.red)),
            if (_result != null) _buildResultsCard(_result!),
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
          children: [
            Text('Total Shots: ${result['total_shots']}'),
            Text('Correct Shots: ${result['correct_shots']}'),
            Text('Processing Time: ${result['processing_time_seconds']}s'),
          ],
        ),
      ),
    );
  }
}
```

### Usage Examples

```dart
import 'package:flutter/material.dart';
import 'package:file_picker/file_picker.dart';
import 'video_processing_screen.dart';

class HomeScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Tennis Trainer')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            ElevatedButton(
              onPressed: () async {
                FilePickerResult? result = await FilePicker.platform.pickFiles(
                  type: FileType.video,
                );
                
                if (result != null && result.files.single.path != null) {
                  // Use WebSocket method
                  Navigator.push(
                    context,
                    MaterialPageRoute(
                      builder: (context) => VideoProcessingScreen(
                        videoFile: File(result.files.single.path!),
                      ),
                    ),
                  );
                }
              },
              child: const Text('Process with WebSocket (Real-time)'),
            ),
            const SizedBox(height: 16),
            ElevatedButton(
              onPressed: () async {
                FilePickerResult? result = await FilePicker.platform.pickFiles(
                  type: FileType.video,
                );
                
                if (result != null && result.files.single.path != null) {
                  // Use HTTP polling method
                  Navigator.push(
                    context,
                    MaterialPageRoute(
                      builder: (context) => VideoProcessingScreenHTTP(
                        videoFile: File(result.files.single.path!),
                      ),
                    ),
                  );
                }
              },
              child: const Text('Process with HTTP Endpoints (Polling)'),
            ),
          ],
        ),
      ),
    );
  }
}
```

## Quick Comparison

| Feature | WebSocket | HTTP Endpoints |
|---------|-----------|----------------|
| **Real-time updates** | ✅ Yes | ❌ Polling required |
| **Server load** | Lower | Higher (polling) |
| **Implementation** | More complex | Simpler |
| **Network usage** | Lower | Higher |
| **Fallback** | Can fallback to HTTP | Always available |
| **Best for** | Production apps | Simple apps, fallback |

## API Endpoints Reference

### 1. Upload Video (Returns Instantly)
```dart
POST /api/analyze
Content-Type: multipart/form-data
Body: { video: File }

Response (202 Accepted - Returns instantly):
{
  "success": true,
  "id": "abc123",                    // Use this ID to get full details
  "session_id": "abc123",            // Alias for compatibility
  "event": "job_complete",           // Event name to listen for
  "status": "pending",
  "message": "Video uploaded successfully...",
  "websocket": {
    "room": "abc123",
    "subscribe_event": "subscribe",
    "listen_event": "job_complete",  // Listen for this event to get full details
    "progress_event": "progress_update"
  },
  "http_endpoint": "/api/status/abc123",  // Use this endpoint to get full details
  "estimated_processing_time_seconds": 120
}
```

**Key Points:**
- Response returns instantly (< 1 second)
- Contains `id` (session_id) - use this to get full details
- Contains `event` name (`job_complete`) - listen for this event
- Use `id` with WebSocket or HTTP endpoint to get full result data

### 2. Get Job Status
```dart
GET /api/status/{session_id}

Response (200 OK):
{
  "session_id": "abc123",
  "status": "processing" | "completed" | "failed",
  "progress": 45,
  "message": "Processing video...",
  "result": { ... } // Only when status is "completed"
}
```

### 3. Get Queue Status
```dart
GET /api/queue

Response (200 OK):
{
  "queue_status": {
    "pending": 2,
    "processing": 3,
    "completed": 10,
    "failed": 0,
    "active_jobs": 5,
    "max_concurrent_jobs": 3,
    "available_slots": 0
  }
}
```

### 4. Health Check
```dart
GET /api/health

Response (200 OK):
{
  "status": "healthy",
  "device_available": {
    "mps": false,
    "cuda": false,
    "cpu": true
  },
  "processing": {
    "active_jobs": 2,
    "max_concurrent": 3
  }
}
```

## WebSocket Events

### Client → Server Events

1. **`subscribe`** - Subscribe to session updates
   ```dart
   socket.emit('subscribe', {'session_id': 'abc123'});
   ```

2. **`unsubscribe`** - Unsubscribe from session
   ```dart
   socket.emit('unsubscribe', {'session_id': 'abc123'});
   ```

### Server → Client Events

1. **`connected`** - Connection established
   ```dart
   socket.on('connected', (data) {
     print('Connected: ${data['message']}');
   });
   ```

2. **`progress_update`** - Progress update
   ```dart
   socket.on('progress_update', (data) {
     // data: {session_id, progress, message, status}
   });
   ```

3. **`job_complete`** - Processing complete
   ```dart
   socket.on('job_complete', (data) {
     // data: {session_id, status, progress, result}
   });
   ```

4. **`job_error`** - Processing error
   ```dart
   socket.on('job_error', (data) {
     // data: {session_id, status, error, message}
   });
   ```

## Hybrid Approach (WebSocket with HTTP Fallback)

```dart
class VideoProcessingServiceHybrid {
  final VideoProcessingService websocketService;
  final VideoProcessingServiceHTTP httpService;
  
  VideoProcessingServiceHybrid({required String baseUrl})
      : websocketService = VideoProcessingService(baseUrl: baseUrl),
        httpService = VideoProcessingServiceHTTP(baseUrl: baseUrl);
  
  Future<Map<String, dynamic>> processVideo(File videoFile) async {
    try {
      // Try WebSocket first
      String sessionId = await websocketService.uploadVideo(videoFile);
      websocketService.connectWebSocket();
      websocketService.subscribeToSession(sessionId);
      
      // Wait for completion with WebSocket
      return await websocketService.waitForCompletion(sessionId);
    } catch (e) {
      // Fallback to HTTP polling
      print('WebSocket failed, using HTTP fallback: $e');
      String sessionId = await httpService.uploadVideo(videoFile);
      return await httpService.waitForCompletion(sessionId);
    }
  }
}
```

## Error Handling

```dart
// Handle connection errors
socket!.onError((error) {
  print('WebSocket error: $error');
  // Fallback to HTTP polling
  _startPolling();
});

// Handle disconnections
socket!.onDisconnect((_) {
  print('Disconnected, attempting reconnect...');
  // Implement reconnection logic
});
```

## Best Practices

1. **Always dispose WebSocket connections** in `dispose()` method
2. **Handle reconnection** if connection drops
3. **Use HTTP polling as fallback** if WebSocket fails
4. **Show loading states** while uploading
5. **Display progress** to keep users informed
6. **Handle errors gracefully** with user-friendly messages

## Testing

Test with your Render backend URL:

```dart
final service = VideoProcessingService(
  baseUrl: 'https://your-app.onrender.com',
);
```

Make sure CORS is enabled on your backend (already configured in `app.py`).

