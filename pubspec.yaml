import 'dart:async';
import 'dart:convert';
import 'dart:io';
import 'dart:math';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:flutter_nearby_connections/flutter_nearby_connections.dart';
import 'package:local_auth/local_auth.dart';
import 'package:encrypt/encrypt.dart' as encrypt;
import 'package:image_picker/image_picker.dart';

// --- Encryption Service ---
class EncryptionService {
  static final _key = encrypt.Key.fromUtf8('my32lengthsupersecretnexuskey!!');
  static final _iv = encrypt.IV.fromLength(16);
  static final _encrypter = encrypt.Encrypter(encrypt.AES(_key));

  static String encryptText(String text) {
    final encrypted = _encrypter.encrypt(text, iv: _iv);
    return encrypted.base64;
  }

  static String decryptText(String encryptedText) {
    try {
      final encryptedData = encrypt.Encrypted.fromBase64(encryptedText);
      return _encrypter.decrypt(encryptedData, iv: _iv);
    } catch (e) {
      print("Decryption Error: $e");
      return "Unable to decrypt message.";
    }
  }
}

// --- Data Models ---
enum MessageType { text, image }

class ChatMessage {
  final String messageId;
  final String originalSenderId;
  final String originalSenderName;
  final String destinationId;
  final MessageType type;
  final String content;
  final DateTime timestamp;

  ChatMessage({
    required this.messageId,
    required this.originalSenderId,
    required this.originalSenderName,
    required this.destinationId,
    required this.type,
    required this.content,
    required this.timestamp,
  });

  Map<String, dynamic> toJson() => {
        'messageId': messageId,
        'originalSenderId': originalSenderId,
        'originalSenderName': originalSenderName,
        'destinationId': destinationId,
        'type': type.toString(),
        'content': content,
        'timestamp': timestamp.toIso8601String(),
      };

  factory ChatMessage.fromJson(Map<String, dynamic> json) => ChatMessage(
        messageId: json['messageId'],
        originalSenderId: json['originalSenderId'],
        originalSenderName: json['originalSenderName'],
        destinationId: json['destinationId'],
        type: MessageType.values.firstWhere((e) => e.toString() == json['type']),
        content: json['content'],
        timestamp: DateTime.parse(json['timestamp']),
      );
}

// --- Main App Entry Point ---
void main() {
  runApp(const NexusApp());
}

class NexusApp extends StatelessWidget {
  const NexusApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Nexus',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        fontFamily: 'Tajawal',
        brightness: Brightness.dark,
        // --- NEW COLOR SCHEME ---
        primaryColor: const Color(0xFF1B263B), // Charcoal Grey
        scaffoldBackgroundColor: const Color(0xFF0D1B2A), // Darker Charcoal
        cardColor: const Color(0xFF1B263B),
        colorScheme: const ColorScheme.dark(
          primary: Color(0xFF40E0D0), // Cyan Blue (for buttons and highlights)
          secondary: Color(0xFF40E0D0), // Cyan Blue
          onPrimary: Colors.black,
          onSecondary: Colors.black,
        ),
        textTheme: const TextTheme(
          bodyLarge: TextStyle(color: Color(0xFFE0FBFC), fontSize: 16),
          bodyMedium: TextStyle(color: Color(0xFF98C1D9), fontSize: 14),
          titleLarge: TextStyle(color: Colors.white, fontWeight: FontWeight.bold, fontSize: 26),
          titleMedium: TextStyle(color: Colors.white, fontWeight: FontWeight.bold),
        ),
        elevatedButtonTheme: ElevatedButtonThemeData(
          style: ElevatedButton.styleFrom(
            backgroundColor: const Color(0xFF40E0D0), // Cyan Blue
            foregroundColor: Colors.black, // Black text on cyan buttons
            shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(30)),
            padding: const EdgeInsets.symmetric(horizontal: 20, vertical: 12),
          ),
        ),
        appBarTheme: const AppBarTheme(
          backgroundColor: Color(0xFF0D1B2A),
          elevation: 0,
          titleTextStyle: TextStyle(fontFamily: 'Tajawal', fontSize: 22, fontWeight: FontWeight.bold),
        ),
      ),
      home: const AuthenticationPage(),
    );
  }
}

// --- Authentication Page ---
class AuthenticationPage extends StatefulWidget {
  const AuthenticationPage({super.key});
  @override
  State<AuthenticationPage> createState() => _AuthenticationPageState();
}

class _AuthenticationPageState extends State<AuthenticationPage> {
  final LocalAuthentication auth = LocalAuthentication();

  @override
  void initState() {
    super.initState();
    _authenticate();
  }

  Future<void> _authenticate() async {
    try {
      bool authenticated = await auth.authenticate(
        localizedReason: 'افتح "Nexus" للمتابعة',
        options: const AuthenticationOptions(stickyAuth: true, biometricOnly: false),
      );
      if (authenticated && mounted) {
        Navigator.of(context).pushReplacement(
          PageRouteBuilder(
            pageBuilder: (context, animation, secondaryAnimation) => const DeviceDiscoveryPage(),
            transitionsBuilder: (context, animation, secondaryAnimation, child) {
              return FadeTransition(opacity: animation, child: child);
            },
          ),
        );
      }
    } on PlatformException catch (e) {
      print("Authentication Error: $e");
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Icon(Icons.shield_moon_rounded, size: 100, color: Theme.of(context).colorScheme.secondary),
            const SizedBox(height: 30),
            const Text('التحقق مطلوب', style: TextStyle(fontSize: 24, fontWeight: FontWeight.bold)),
            const SizedBox(height: 30),
            ElevatedButton.icon(
              onPressed: _authenticate,
              icon: const Icon(Icons.fingerprint),
              label: const Text('افتح التطبيق'),
            ),
          ],
        ),
      ),
    );
  }
}

// --- Device Discovery Page ---
class DeviceDiscoveryPage extends StatefulWidget {
  const DeviceDiscoveryPage({super.key});
  @override
  State<DeviceDiscoveryPage> createState() => _DeviceDiscoveryPageState();
}

class _DeviceDiscoveryPageState extends State<DeviceDiscoveryPage> {
  late NearbyService nearbyService;
  late StreamSubscription _stateSubscription;
  late StreamSubscription _dataSubscription;
  final List<Device> _devices = [];
  final Map<String, List<ChatMessage>> _chatHistory = {};
  String _userName = 'مستخدم-${Random().nextInt(1000)}';
  bool _isScanning = false;
  final Set<String> _processedMessageIds = {};

  @override
  void initState() {
    super.initState();
    _initNearbyService();
  }

  @override
  void dispose() {
    _stateSubscription.cancel();
    _dataSubscription.cancel();
    super.dispose();
  }

  void _initNearbyService() async {
    nearbyService = NearbyService();
    await nearbyService.init(
      serviceType: 'nexus-chat', // Changed service type to match new name
      deviceName: _userName,
      strategy: Strategy.P2P_CLUSTER,
      callback: (isRunning) async {
        if (isRunning) {
          await nearbyService.stopBrowsingForPeers();
          await Future.delayed(const Duration(microseconds: 200));
          await nearbyService.startBrowsingForPeers();
          setState(() => _isScanning = true);
        }
      },
    );

    _stateSubscription = nearbyService.stateChangedSubscription(callback: (devicesList) {
      setState(() {
        _devices.clear();
        _devices.addAll(devicesList);
      });
    });

    _dataSubscription = nearbyService.dataReceivedSubscription(callback: (data) {
      final encryptedJsonString = utf8.decode(data.message);
      final decryptedJsonString = EncryptionService.decryptText(encryptedJsonString);
      final messageJson = jsonDecode(decryptedJsonString);
      final message = ChatMessage.fromJson(messageJson);

      if (_processedMessageIds.contains(message.messageId)) return;
      _processedMessageIds.add(message.messageId);

      if (message.destinationId == nearbyService.myDevice.deviceId) {
        setState(() {
          final chatPartnerId = message.originalSenderId;
          if (!_chatHistory.containsKey(chatPartnerId)) {
            _chatHistory[chatPartnerId] = [];
          }
          _chatHistory[chatPartnerId]!.add(message);
        });
        ScaffoldMessenger.of(context).showSnackBar(SnackBar(
          content: Text('رسالة جديدة من ${message.originalSenderName}'),
          backgroundColor: Theme.of(context).colorScheme.secondary,
          behavior: SnackBarBehavior.floating,
        ));
      } else {
        for (var device in _devices.where((d) => d.state == SessionState.connected && d.deviceId != data.deviceId)) {
          nearbyService.sendMessage(device.deviceId, data.message);
        }
      }
    });
  }

  void _sendMessage(String destinationId, MessageType type, String content) {
    final message = ChatMessage(
      messageId: Random().nextInt(999999).toString(),
      originalSenderId: nearbyService.myDevice.deviceId,
      originalSenderName: _userName,
      destinationId: destinationId,
      type: type,
      content: content,
      timestamp: DateTime.now(),
    );

    final jsonString = jsonEncode(message.toJson());
    final encryptedString = EncryptionService.encryptText(jsonString);
    final rawEncryptedMessage = utf8.encode(encryptedString);

    setState(() {
      final chatPartnerId = destinationId;
      if (!_chatHistory.containsKey(chatPartnerId)) {
        _chatHistory[chatPartnerId] = [];
      }
      _chatHistory[chatPartnerId]!.add(message);
    });

    for (var device in _devices.where((d) => d.state == SessionState.connected)) {
      nearbyService.sendMessage(device.deviceId, rawEncryptedMessage);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        centerTitle: true,
        title: const Text('Nexus'),
        actions: [
          IconButton(
            icon: const Icon(Icons.edit_note),
            onPressed: _showUsernameDialog,
            tooltip: 'تغيير اسمك',
          ),
        ],
      ),
      body: Column(
        children: [
          Padding(
            padding: const EdgeInsets.all(16.0),
            child: Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                Text('المستخدمون المتاحون', style: Theme.of(context).textTheme.titleLarge),
                if (_isScanning) const ScanAnimation(),
              ],
            ),
          ),
          Expanded(
            child: _devices.isEmpty
                ? _buildEmptyState()
                : AnimatedList(
                    initialItemCount: _devices.length,
                    itemBuilder: (context, index, animation) {
                      return SizeTransition(
                        sizeFactor: animation,
                        child: _buildDeviceListItem(_devices[index]),
                      );
                    },
                  ),
          ),
        ],
      ),
    );
  }

  Widget _buildEmptyState() {
    return Center(
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          Icon(Icons.wifi_off_rounded, size: 100, color: Theme.of(context).cardColor),
          const SizedBox(height: 20),
          const Text('لا يوجد مستخدمون في النطاق', style: TextStyle(fontSize: 18)),
          const SizedBox(height: 8),
          Text(
            'تأكد من أن أصدقاءك يستخدمون التطبيق بالقرب منك',
            textAlign: TextAlign.center,
            style: Theme.of(context).textTheme.bodyMedium,
          ),
        ],
      ),
    );
  }

  Widget _buildDeviceListItem(Device device) {
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 16.0, vertical: 6.0),
      child: Card(
        elevation: 4,
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(15)),
        child: ListTile(
          leading: CircleAvatar(
            backgroundColor: _getStatusColor(device.state),
            child: _getDeviceIcon(device.state),
          ),
          title: Text(device.deviceName, style: Theme.of(context).textTheme.titleMedium),
          subtitle: Text(_getStatusText(device.state), style: TextStyle(color: _getStatusColor(device.state))),
          trailing: _buildActionButton(device),
        ),
      ),
    );
  }

  Icon _getDeviceIcon(SessionState state) {
    switch (state) {
      case SessionState.connected: return const Icon(Icons.link, color: Colors.black);
      case SessionState.connecting: return const Icon(Icons.hourglass_top_rounded, color: Colors.black);
      case SessionState.notConnected: return const Icon(Icons.person_search, color: Colors.black);
    }
  }

  String _getStatusText(SessionState state) {
    switch (state) {
      case SessionState.connected: return 'متصل';
      case SessionState.connecting: return 'جاري الاتصال...';
      case SessionState.notConnected: return 'متاح';
    }
  }

  Color _getStatusColor(SessionState state) {
    switch (state) {
      case SessionState.connected: return Theme.of(context).colorScheme.primary; // Cyan
      case SessionState.connecting: return Colors.orange.shade400;
      case SessionState.notConnected: return Colors.grey.shade600;
    }
  }

  Widget? _buildActionButton(Device device) {
    if (device.state == SessionState.notConnected) {
      return ElevatedButton(
        onPressed: () => nearbyService.invitePeer(deviceID: device.deviceId, deviceName: _userName),
        child: const Text('اتصال'),
      );
    }

    if (device.state == SessionState.connecting) {
      return const SizedBox(width: 24, height: 24, child: CircularProgressIndicator(strokeWidth: 3));
    }

    return ElevatedButton(
      onPressed: () {
        Navigator.push(
          context,
          MaterialPageRoute(
            builder: (context) => ChatPage(
              localUsername: _userName,
              peerDevice: device,
              chatHistory: _chatHistory[device.deviceId] ?? [],
              onSendMessage: (type, content) => _sendMessage(device.deviceId, type, content),
            ),
          ),
        );
      },
      child: const Text('محادثة'),
    );
  }

  void _showUsernameDialog() {
    final TextEditingController controller = TextEditingController(text: _userName);
    showDialog(
      context: context,
      builder: (context) {
        return AlertDialog(
          backgroundColor: Theme.of(context).cardColor,
          title: const Text('تغيير اسمك'),
          content: TextField(controller: controller, autofocus: true, style: const TextStyle(color: Colors.white)),
          actions: [
            TextButton(onPressed: () => Navigator.of(context).pop(), child: const Text('إلغاء')),
            ElevatedButton(
              onPressed: () {
                if (controller.text.isNotEmpty) {
                  setState(() => _userName = controller.text);
                  nearbyService.stopBrowsingForPeers();
                  nearbyService.stopAdvertisingPeer();
                  _initNearbyService();
                  Navigator.of(context).pop();
                }
              },
              child: const Text('حفظ'),
            ),
          ],
        );
      },
    );
  }
}

// --- Chat Page ---
class ChatPage extends StatefulWidget {
  final String localUsername;
  final Device peerDevice;
  final List<ChatMessage> chatHistory;
  final Function(MessageType, String) onSendMessage;

  const ChatPage({
    super.key,
    required this.localUsername,
    required this.peerDevice,
    required this.chatHistory,
    required this.onSendMessage,
  });

  @override
  State<ChatPage> createState() => _ChatPageState();
}

class _ChatPageState extends State<ChatPage> {
  final TextEditingController _messageController = TextEditingController();
  final ScrollController _scrollController = ScrollController();
  final ImagePicker _picker = ImagePicker();

  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addPostFrameCallback((_) => _scrollToBottom());
  }
  
  @override
  void didUpdateWidget(covariant ChatPage oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (widget.chatHistory.length > oldWidget.chatHistory.length) {
      _scrollToBottom();
    }
  }

  void _sendMessage() {
    if (_messageController.text.trim().isNotEmpty) {
      widget.onSendMessage(MessageType.text, _messageController.text.trim());
      _messageController.clear();
      _scrollToBottom();
    }
  }

  Future<void> _sendImage() async {
    final XFile? image = await _picker.pickImage(source: ImageSource.gallery, imageQuality: 50);
    if (image != null) {
      final bytes = await image.readAsBytes();
      final base64Image = base64Encode(bytes);
      widget.onSendMessage(MessageType.image, base64Image);
      _scrollToBottom();
    }
  }

  void _scrollToBottom() {
    if (_scrollController.hasClients) {
      _scrollController.animateTo(_scrollController.position.maxScrollExtent, duration: const Duration(milliseconds: 300), curve: Curves.easeOut);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text(widget.peerDevice.deviceName)),
      body: Container(
        decoration: BoxDecoration(
          gradient: LinearGradient(
            colors: [Theme.of(context).scaffoldBackgroundColor, const Color(0xFF1B263B)],
            begin: Alignment.topCenter,
            end: Alignment.bottomCenter,
          ),
        ),
        child: Column(
          children: [
            Expanded(
              child: ListView.builder(
                controller: _scrollController,
                padding: const EdgeInsets.all(16),
                itemCount: widget.chatHistory.length,
                itemBuilder: (context, index) {
                  final message = widget.chatHistory[index];
                  final isMe = message.originalSenderName == widget.localUsername;
                  return _buildMessageBubble(message, isMe);
                },
              ),
            ),
            _buildMessageInput(),
          ],
        ),
      ),
    );
  }

  Widget _buildMessageBubble(ChatMessage message, bool isMe) {
    return Align(
      alignment: isMe ? Alignment.centerRight : Alignment.centerLeft,
      child: Card(
        elevation: 5,
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.only(
            topLeft: const Radius.circular(20),
            topRight: const Radius.circular(20),
            bottomLeft: isMe ? const Radius.circular(20) : const Radius.circular(0),
            bottomRight: isMe ? const Radius.circular(0) : const Radius.circular(20),
          ),
        ),
        color: isMe ? Theme.of(context).colorScheme.primary : Theme.of(context).cardColor,
        child: Padding(
          padding: const EdgeInsets.all(12.0),
          child: Column(
            crossAxisAlignment: isMe ? CrossAxisAlignment.end : CrossAxisAlignment.start,
            children: [
              if (!isMe) Text(message.originalSenderName, style: TextStyle(fontWeight: FontWeight.bold, fontSize: 12, color: Colors.white70)),
              const SizedBox(height: 4),
              if (message.type == MessageType.text)
                Text(message.content, style: TextStyle(color: isMe ? Colors.black : Colors.white, fontSize: 16))
              else if (message.type == MessageType.image)
                ClipRRect(
                  borderRadius: BorderRadius.circular(12),
                  child: Image.memory(base64Decode(message.content),
                    width: MediaQuery.of(context).size.width * 0.6,
                  ),
                ),
            ],
          ),
        ),
      ),
    );
  }

  Widget _buildMessageInput() {
    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 8.0, vertical: 12.0),
      decoration: BoxDecoration(color: Theme.of(context).cardColor.withOpacity(0.8)),
      child: SafeArea(
        child: Row(
          children: [
            IconButton(icon: Icon(Icons.attach_file_rounded, color: Colors.white70), onPressed: _sendImage),
            Expanded(
              child: TextField(
                controller: _messageController,
                style: const TextStyle(color: Colors.white),
                decoration: InputDecoration(
                  hintText: 'اكتب رسالتك...',
                  hintStyle: TextStyle(color: Colors.grey[400]),
                  border: InputBorder.none,
                  filled: true,
                  fillColor: Theme.of(context).scaffoldBackgroundColor,
                  contentPadding: const EdgeInsets.symmetric(horizontal: 20, vertical: 14),
                  border: OutlineInputBorder(borderRadius: BorderRadius.circular(30), borderSide: BorderSide.none),
                ),
                onSubmitted: (_) => _sendMessage(),
              ),
            ),
            const SizedBox(width: 8),
            FloatingActionButton(
              mini: true,
              onPressed: _sendMessage,
              backgroundColor: Theme.of(context).colorScheme.secondary,
              child: const Icon(Icons.send_rounded, color: Colors.black, size: 20),
            ),
          ],
        ),
      ),
    );
  }
}

// --- Custom Animation Widget ---
class ScanAnimation extends StatefulWidget {
  const ScanAnimation({super.key});

  @override
  State<ScanAnimation> createState() => _ScanAnimationState();
}

class _ScanAnimationState extends State<ScanAnimation> with SingleTickerProviderStateMixin {
  late AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 1),
    )..repeat(reverse: true);
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return FadeTransition(
      opacity: Tween(begin: 0.5, end: 1.0).animate(
        CurvedAnimation(parent: _controller, curve: Curves.easeInOut),
      ),
      child: Icon(Icons.radar, color: Theme.of(context).colorScheme.secondary),
    );
  }
}
