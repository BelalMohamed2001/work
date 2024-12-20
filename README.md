import 'package:flutter/material.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:connectivity_plus/connectivity_plus.dart'; // To check connectivity
import '../controllers/event_controller.dart';
import '../models/event_model.dart';
import '../helpers/database_helper.dart'; // Import your SQLite helper
import 'gift_list_page.dart';

class EventListPage extends StatefulWidget {
  const EventListPage({super.key});

  @override
  _EventListPageState createState() => _EventListPageState();
}

class _EventListPageState extends State<EventListPage> {
  final EventController _eventController = EventController();
  final User? _currentUser = FirebaseAuth.instance.currentUser;
  List<EventModel> _events = [];
  bool _isLoading = true;
  bool _isAscending = true;
  bool _isOnline = false;

  @override
  void initState() {
    super.initState();
    if (_currentUser != null) {
      _checkConnectivity();
      _loadEvents();
    }
  }

  // Check if the device is online or offline
  void _checkConnectivity() async {
    var connectivityResult = await (Connectivity().checkConnectivity());
    setState(() {
      _isOnline = connectivityResult != ConnectivityResult.none;
    });
  }

  // Load events from Firestore or SQLite based on the connectivity status
  Future<void> _loadEvents() async {
    setState(() => _isLoading = true);

    if (_isOnline) {
      // Fetch from Firestore if online
      _events = await _eventController.fetchEvents(_currentUser!.uid);
      // Optionally, sync the events with SQLite
      for (var event in _events) {
        await DatabaseHelper.instance.insertEvent(event);
      }
    } else {
      // Fetch from SQLite if offline
      _events = await DatabaseHelper.instance.fetchEvents(_currentUser!.uid);
    }

    setState(() => _isLoading = false);
  }

  // Sort events based on selected index and order
  void _sortEvents(int index, bool isAscending) {
    setState(() {
      if (index == 0) {
        _events.sort((a, b) => a.name.toLowerCase().compareTo(b.name.toLowerCase()));
      } else if (index == 1) {
        _events.sort((a, b) => a.category.toLowerCase().compareTo(b.category.toLowerCase()));
      } else {
        _events.sort((a, b) => a.status.toLowerCase().compareTo(b.status.toLowerCase()));
      }

      if (!isAscending) {
        _events = _events.reversed.toList(); // Reverse the order
      }
    });
  }

  // Show dialog to edit or create an event
  Future<void> _editEventDialog(EventModel? event) async {
    TextEditingController nameController = TextEditingController(text: event?.name ?? '');
    TextEditingController categoryController = TextEditingController(text: event?.category ?? '');
    String selectedStatus = event?.status ?? 'Upcoming';

    await showDialog(
      context: context,
      builder: (context) {
        return AlertDialog(
          title: Text(event == null ? 'Add Event' : 'Edit Event'),
          content: SingleChildScrollView(
            child: Column(
              mainAxisSize: MainAxisSize.min,
              children: [
                TextField(
                  controller: nameController,
                  decoration: const InputDecoration(labelText: 'Event Name'),
                ),
                TextField(
                  controller: categoryController,
                  decoration: const InputDecoration(labelText: 'Event Category'),
                ),
                DropdownButtonFormField<String>(
                  value: selectedStatus,
                  items: const [
                    DropdownMenuItem(value: 'Upcoming', child: Text('Upcoming')),
                    DropdownMenuItem(value: 'Current', child: Text('Current')),
                    DropdownMenuItem(value: 'Past', child: Text('Past')),
                  ],
                  onChanged: (value) {
                    if (value != null) {
                      selectedStatus = value;
                    }
                  },
                  decoration: const InputDecoration(labelText: 'Event Status'),
                ),
              ],
            ),
          ),
          actions: [
            TextButton(
              onPressed: () => Navigator.pop(context),
              child: const Text('Cancel'),
            ),
            TextButton(
              onPressed: () async {
                if (nameController.text.isEmpty || categoryController.text.isEmpty) {
                  ScaffoldMessenger.of(context).showSnackBar(
                    const SnackBar(content: Text('Please fill in all fields')),
                  );
                  return;
                }

                final updatedEvent = EventModel(
                  id: event?.id ?? '',
                  name: nameController.text,
                  category: categoryController.text,
                  status: selectedStatus,
                  userId: _currentUser!.uid,
                  date: event?.date ?? DateTime.now().toIso8601String(),
                  description: event?.description ?? '',
                );

                if (event == null) {
                  await _eventController.addEvent(updatedEvent);
                  if (!_isOnline) {
                    // Save event locally if offline
                    await DatabaseHelper.instance.insertEvent(updatedEvent);
                  }
                } else {
                  await _eventController.updateEvent(updatedEvent);
                  if (!_isOnline) {
                    // Update event locally if offline
                    await DatabaseHelper.instance.updateEvent(updatedEvent);
                  }
                }

                Navigator.pop(context);
                _loadEvents(); // Reload the events
              },
              child: const Text('Save'),
            ),
          ],
        );
      },
    );
  }

  // Delete an event
  void _deleteEvent(String id) async {
    await _eventController.deleteEvent(id);
    if (!_isOnline) {
      // Delete event locally if offline
      await DatabaseHelper.instance.deleteEvent(id);
    }
    _loadEvents();
  }

  // Navigate to the gift list page
  void _goToGiftListPage(EventModel event) {
    Navigator.push(
      context,
      MaterialPageRoute(
        builder: (context) => GiftListPage(eventId: event.id), // Pass the event id to GiftListPage
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Event List'),
        backgroundColor: Colors.pinkAccent,
      ),
      body: _isLoading
          ? const Center(child: CircularProgressIndicator())
          : _events.isEmpty
              ? const Center(child: Text('No events found.'))
              : Column(
                  children: [
                    Padding(
                      padding: const EdgeInsets.all(8.0),
                      child: Row(
                        mainAxisAlignment: MainAxisAlignment.spaceAround,
                        children: [
                          Tooltip(
                            message: 'Sort by Name',
                            child: IconButton(
                              icon: const Icon(Icons.sort_by_alpha),
                              onPressed: () {
                                _sortEvents(0, !_isAscending);
                                setState(() {
                                  _isAscending = !_isAscending;
                                });
                              },
                            ),
                          ),
                          Tooltip(
                            message: 'Sort by Category',
                            child: IconButton(
                              icon: const Icon(Icons.category),
                              onPressed: () {
                                _sortEvents(1, !_isAscending);
                                setState(() {
                                  _isAscending = !_isAscending;
                                });
                              },
                            ),
                          ),
                          Tooltip(
                            message: 'Sort by Status',
                            child: IconButton(
                              icon: const Icon(Icons.timeline),
                              onPressed: () {
                                _sortEvents(2, !_isAscending);
                                setState(() {
                                  _isAscending = !_isAscending;
                                });
                              },
                            ),
                          ),
                        ],
                      ),
                    ),
                    Expanded(
                      child: ListView.builder(
                        itemCount: _events.length,
                        itemBuilder: (context, index) {
                          final event = _events[index];
                          return Card(
                            margin: const EdgeInsets.symmetric(vertical: 8, horizontal: 16),
                            child: ListTile(
                              title: Text(event.name),
                              subtitle: Text('${event.category} - ${event.status}'),
                              onTap: () => _goToGiftListPage(event), // Navigate to gift list page
                              trailing: Row(
                                mainAxisSize: MainAxis

3/3









