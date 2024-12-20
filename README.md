import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:connectivity_plus/connectivity_plus.dart';
import 'dart:convert';
import '../controllers/database_helper.dart';

class SyncManager {
  final FirebaseFirestore _firestore = FirebaseFirestore.instance;

  // Method to check online/offline status
  Future<bool> isOnline() async {
    var connectivityResult = await (Connectivity().checkConnectivity());
    return connectivityResult != ConnectivityResult.none;
  }

  // Sync Events (With Handling for Deletions)
  Future<void> syncEvents(String userId) async {
    List<Map<String, dynamic>> pendingOperations = await DatabaseHelper.instance.fetchPendingOperations();

    for (var operation in pendingOperations) {
      if (operation['table_name'] == 'events') {
        var eventData = jsonDecode(operation['data']);
        bool synced = false;

        try {
          if (operation['operation'] == 'add') {
            await _firestore.collection('events').add(eventData);
          } else if (operation['operation'] == 'update') {
            await _firestore.collection('events').doc(eventData['id']).set(eventData);
          } else if (operation['operation'] == 'delete') {
            await _firestore.collection('events').doc(eventData['id']).delete();
            await DatabaseHelper.instance.deleteEvent(eventData['id']); // Removing from SQLite
          }
          synced = true;
        } catch (e) {
          print("Error syncing event: $e");
        }

        if (synced) {
          await DatabaseHelper.instance.updatePendingOperationStatus(operation['id'], 'completed');
        }
      }
    }
  }

  // Sync Gifts (With Handling for Deletions)
  Future<void> syncGifts(String eventId) async {
    List<Map<String, dynamic>> pendingOperations = await DatabaseHelper.instance.fetchPendingOperations();

    for (var operation in pendingOperations) {
      if (operation['table_name'] == 'gifts') {
        var giftData = jsonDecode(operation['data']);
        bool synced = false;

        try {
          if (operation['operation'] == 'add') {
            await _firestore.collection('gifts').add(giftData);
          } else if (operation['operation'] == 'update') {
            await _firestore.collection('gifts').doc(giftData['id']).set(giftData);
          } else if (operation['operation'] == 'delete') {
            await _firestore.collection('gifts').doc(giftData['id']).delete();
            await DatabaseHelper.instance.deleteGift(giftData['id']); // Removing from SQLite
          }
          synced = true;
        } catch (e) {
          print("Error syncing gift: $e");
        }

        if (synced) {
          await DatabaseHelper.instance.updatePendingOperationStatus(operation['id'], 'completed');
        }
      }
    }
  }

  // Sync Users (Handling User Sync for add or update operations)
  Future<void> syncUsers() async {
    List<Map<String, dynamic>> pendingOperations = await DatabaseHelper.instance.fetchPendingOperations();

    for (var operation in pendingOperations) {
      if (operation['table_name'] == 'users') {
        var userData = jsonDecode(operation['data']);
        bool synced = false;

        try {
          if (operation['operation'] == 'add') {
            await _firestore.collection('users').add(userData);
          } else if (operation['operation'] == 'update') {
            await _firestore.collection('users').doc(userData['uid']).set(userData);
          }
          synced = true;
        } catch (e) {
          print("Error syncing user: $e");
        }

        if (synced) {
          await DatabaseHelper.instance.updatePendingOperationStatus(operation['id'], 'completed');
        }
      }
    }
  }

  // Sync all data (Users, Events, Gifts) when online
  Future<void> syncAllData() async {
    if (await isOnline()) {
      String userId = "user_id_example"; // You would replace this with actual user ID

      // Syncing events and gifts
      await syncEvents(userId);
      await syncGifts(userId);

      // You can also sync users if needed
      await syncUsers();
    } else {
      print("Device is offline. Syncing will resume once online.");
    }
  }
}
