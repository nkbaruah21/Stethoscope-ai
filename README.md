 pubspec.yaml

name: stethoscope_app
description: "A stethoscope diagnostic assistant app."
publish_to: 'none' 
version: 1.0.0+1

environment:
  sdk: '>=3.2.3 <4.0.0'

dependencies:
  flutter:
    sdk: flutter
  
  # For Bluetooth Low Energy
  flutter_blue_plus: ^1.31.18 

  # For Audio Recording
  record: ^5.0.4
  path_provider: ^2.1.2
  permission_handler: ^11.3.1

  # For AI Model Inference
  tflite_flutter: ^0.10.4

  # For Local Database (SQLite)
  drift: ^2.16.0
  sqlite3_flutter_libs: ^0.5.21
  path: ^1.9.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  
  # Drift code generator
  drift_dev: ^2.16.0
  build_runner: ^2.4.8

flutter:
  uses-material-design: true

  # Make sure to add your AI model file in an assets folder
  assets:
    - assets/models/
// lib/main.dart

import 'package:flutter/material.dart';
import 'package:stethoscope_app/screens/home_screen.dart'; // We will create this next

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Stethoscope App',
      theme: ThemeData(
        primarySwatch: Colors.blue,
        visualDensity: VisualDensity.adaptivePlatformDensity,
        useMaterial3: true,
      ),
      // For simplicity, we'll start at the home screen.
      // A proper app would have a login/splash screen system.
      home: HomeScreen(),
    );
  }
}
// lib/screens/home_screen.dart

import 'package:flutter/material.dart';
import 'package:stethoscope_app/screens/record_screen.dart';

class HomeScreen extends StatelessWidget {
  const HomeScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Dashboard'),
        actions: [
          // Icon to show Bluetooth connection status
          IconButton(
            icon: const Icon(Icons.bluetooth_disabled, color: Colors.red),
            onPressed: () {
              // TODO: Navigate to Bluetooth connection screen
              print("Connecting to stethoscope...");
            },
          ),
        ],
      ),
      body: ListView.builder(
        itemCount: 5, // Placeholder for patient list
        itemBuilder: (context, index) {
          return ListTile(
            title: Text('Patient #${index + 1}'),
            subtitle: const Text('Age: 34, Last checked: 2025-07-28'),
            leading: const Icon(Icons.person_outline),
            onTap: () {
              Navigator.push(
                context,
                MaterialPageRoute(
                  builder: (context) => RecordScreen(patientId: index + 1),
                ),
              );
            },
          );
        },
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          // TODO: Add new patient logic
        },
        child: const Icon(Icons.add),
        tooltip: 'Add Patient',
      ),
    );
  }
}
// lib/services/bluetooth_service.dart

import 'package:flutter_blue_plus/flutter_blue_plus.dart';
import 'dart:async';

class BluetoothService {
  Stream<List<ScanResult>> get scanResults => FlutterBluePlus.scanResults;
  Stream<BluetoothConnectionState> state(BluetoothDevice device) => device.connectionState;

  // Start scanning for devices
  void startScan() {
    FlutterBluePlus.startScan(timeout: const Duration(seconds: 5));
  }

  // Stop scanning
  void stopScan() {
    FlutterBluePlus.stopScan();
  }

  // Connect to a device
  Future<void> connectToDevice(BluetoothDevice device) async {
    try {
      await device.connect(timeout: const Duration(seconds: 15));
      print("Connected to ${device.localName}");
      // TODO: Discover services and characteristics provided by the Stethoscope SDK
    } catch (e) {
      print("Error connecting to device: $e");
    }
  }

  // Disconnect
  void disconnectFromDevice(BluetoothDevice device) {
    device.disconnect();
    print("Disconnected from ${device.localName}");
  }
}
// lib/services/audio_recorder_service.dart

import 'package:record/record.dart';
import 'package:path_provider/path_provider.dart';
import 'package:permission_handler/permission_handler.dart';

class AudioRecorderService {
  final AudioRecorder _recorder = AudioRecorder();

  Future<bool> get _isPermissionGranted async => await Permission.microphone.isGranted;

  // Request recording permission
  Future<void> requestPermission() async {
    if (!await _isPermissionGranted) {
      await Permission.microphone.request();
    }
  }

  // Start recording
  Future<String?> startRecording() async {
    await requestPermission();
    if (await _isPermissionGranted) {
      final directory = await getApplicationDocumentsDirectory();
      final path = '${directory.path}/recording_${DateTime.now().millisecondsSinceEpoch}.m4a';
      
      await _recorder.start(const RecordConfig(), path: path);
      print("Recording started, saving to: $path");
      return path;
    } else {
      print("Microphone permission not granted.");
      return null;
    }
  }

  // Stop recording
  Future<String?> stopRecording() async {
    final path = await _recorder.stop();
    print("Recording stopped. File saved at: $path");
    return path;
  }

  void dispose() {
    _recorder.dispose();
  }
}
// lib/services/ai_model_service.dart

import 'package:tflite_flutter/tflite_flutter.dart';

class AiModelService {
  late Interpreter _interpreter;

  // Load the model from assets
  Future<void> loadModel() async {
    try {
      _interpreter = await Interpreter.fromAsset('assets/models/heart_murmur_model.tflite');
      print("TFLite model loaded successfully.");
    } catch (e) {
      print("Failed to load model: $e");
    }
  }
  
  // Run inference
  String runInference(List<dynamic> inputData) {
    // This is a placeholder for your actual input shape
    // Example: A spectrogram of shape [1, 96, 64, 1]
    // final input = [inputData]; // Reshape if necessary

    // Define output tensor shape
    // Example: A classification output of shape [1, 2] for [normal, abnormal]
    var output = List.filled(1 * 2, 0).reshape([1, 2]);

    // Run the model
    _interpreter.run(inputData, output);

    // Process the output
    double normalProb = output[0][0];
    double abnormalProb = output[0][1];
    
    print("Inference Result - Normal: $normalProb, Abnormal: $abnormalProb");

    if (abnormalProb > 0.75) { // Example threshold
      return "Abnormal sound detected (Possible murmur)";
    } else {
      return "Normal sound detected";
    }
  }
  
  void dispose() {
    _interpreter.close();
  }
}
// lib/database/database.dart

import 'dart:io';
import 'package:drift/drift.dart';
import 'package:drift/native.dart';
import 'package:path_provider/path_provider.dart';
import 'package:path/path.dart' as p;

part 'database.g.dart'; // This will be generated by build_runner

// Define the tables
class Patients extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get name => text().withLength(min: 1, max: 50)();
  IntColumn get age => integer()();
}

class Recordings extends Table {
  IntColumn get id => integer().autoIncrement()();
  IntColumn get patientId => integer().references(Patients, #id)();
  TextColumn get audioFilePath => text()();
  TextColumn get diagnosis => text()();
  DateTimeColumn get recordedAt => dateTime()();
}

// Define the database class
@DriftDatabase(tables: [Patients, Recordings])
class AppDatabase extends _$AppDatabase {
  AppDatabase() : super(_openConnection());

  @override
  int get schemaVersion => 1;

  // Methods to interact with the database
  Future<int> addRecording(RecordingsCompanion entry) {
    return into(recordings).insert(entry);
  }

  Future<List<Recording>> getRecordingsForPatient(int patientId) {
    return (select(recordings)..where((tbl) => tbl.patientId.equals(patientId))).get();
  }
}

LazyDatabase _openConnection() {
  return LazyDatabase(() async {
    final dbFolder = await getApplicationDocumentsDirectory();
    final file = File(p.join(dbFolder.path, 'db.sqlite'));
    return NativeDatabase.createInBackground(file);
  });
}
