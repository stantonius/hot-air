---
title: "Managing Geofence events with Riverpod"
description: "How I used the Flutter Riverpod library to manage Geofence events"
layout: post
toc: true
comments: true
hide: false
categories: [Flutter, Riverpod]
---

# Managing Geofence events with Riverpod

As described in the blog [Proximity-triggered LEDs](https://stantonius.github.io/home/iot/microcomputing/flutter/2021/12/22/proximity_leds_main.html), my Android phone is able to advertize as a beacon that can be used by other systems in the house to track its proximity.

However, given that I try to be very security conscious, one of the problems I came across is how to prevent the beacon from broadcasting away from our house. Additionally, restricting Flutter events only when in a specific location will also help with battery life.

I was able to use Riverpod state management library to turn on/off Bluetooth Low Energy advertising and MQTT connections when I enter/leave the zone around my house.

## Code
The code for this app is a work in progress, but can be found [here](https://github.com/stantonius/flutter_smart_home_app)

Note this project used the awesome [geofence_service](https://pub.dev/packages/geofence_service) plugin and a lot of the code is borrowed from the examples in this library.

## Solution
The Riverpod code was integrated into the `geofence.dart` file to solution the following tasks:
1. Update a state when geofence status changes
2. Turn BLE and MQTT on if geofence status is "ENTER", and turn them off if status is "LEAVE"

### 1. Stream Changes <> State
Near the top of the `geofence.dart` file, you can see we have made 3 `StreamController`s. 

```dart

// example of one of the StreamController

StreamController<GeofenceStatus> _geofenceStreamController = StreamController<GeofenceStatus>();
```

> This is where I got caught up. `StreamController` is not a Riverpod class, and yet I was looking for solutions to this state problem exclusively within Riverpod. However Riverpod is just an (excellent) tool to facilitate state management - it doesn't have all the answers (it wasn't designed to replace all base classes provided by Dart)

`StreamController` is effectively a *container* that holds the stream values. But on its own it is useless. We need to make a connection between when the phone signals there has been a geofence change and the `StreamController`. This is done in 2 parts:

1. Create a `_onGeofenceStatusChanged()` callback function that *binds the geofence change to the `StreamController` sink*. In this case we are only storing the `GeofenceStatus`, but we could store others (such as distance to geofence).

```dart
Future<void> _onGeofenceStatusChanged(
    Geofence geofence,
    GeofenceRadius geofenceRadius,
    GeofenceStatus geofenceStatus,
    Location location) async {
  print('geofence: $geofence');
  print('geofenceRadius: $geofenceRadius');
  print('geofenceStatus: ${geofenceStatus.toString()}');
  _geofenceStreamController.sink.add(geofenceStatus); 	//this is the binding
}
```

2.  Use the Riverpod `StreamProvider` class, to *link* any changes from the stream *to any ref that is watching for changes*

```dart
final geofenceStreamProvider = AutoDisposeStreamProvider<GeofenceStatus>((ref) {
	ref.onDispose(() {
		_geofenceStreamController.close();
	});
	// update another state object
  return _geofenceStreamController.stream;
});

//....

class GeofenceDetails extends ConsumerWidget {
  const GeofenceDetails({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context, WidgetRef ref) {
	// below watches for any changes via the geofenceStreamProvider
    AsyncValue geofenceState = ref.watch(geofenceStreamProvider);
	  
	// do something with the asyncvalue
	 
  }
}

```

We now have a system that can update state for every geofence status change. Next is to turn the beacon or MQTT connections on or off depending on the geofence status.

However we must tackle another item first - set the initial state when the app is loaded for the first time. 

### Set the initial Geofence Status

The way we handled this was to always assume the geofence status was `GeofenceStatus.EXIT` until the system overrides it. We used the callback system of the `geofence_service` plugin to set this status via the `.start` future.

```dart
  WidgetsBinding.instance?.addPostFrameCallback((_) {
	// other callback bindings
  _geofenceService.start(_geofenceList).catchError(_onError).whenComplete(
        () => _geofenceStreamController.sink.add(GeofenceStatus.EXIT));
	  	// when the start future is complete, set the geofence state to EXIT
  }
```

Now that we have an initial Geofence status that will update if we are within any of the geofences defined, we can set the BLE and MQTT properties.

### 2. Turn on BLE and MQTT

It turns out that when the app is loaded, the BLE and MQTT states need to be set in an *async* manner. Because we cannot actually use an `async` when setting the app state, we needed another approach.

While not elegent, I settled on calling the `ProviderContainer` object to update the Provider state outside of a `ConsumerWidget` or another Provider. The `ProviderContainer` seems to be a wrapper class that houses all of the Providers and their states for the entire app. 

```dart

// code from main.dart outlining the container for providers used in this app

final container = ProviderContainer();

// permissions class
final permissions = SmartHomePermissions();

void main() {
  runApp(UncontrolledProviderScope(container: container, child: MyApp()));
}
```

[The author of Riverpod suggests](https://github.com/rrousselGit/river_pod/issues/295#issuecomment-770368500) this is a possible (yet not recommended) solution to get access to the Provider state elsewhere in the UI (they go on to explain how you might approach this differently but I couldn't wrap my head around this, and my solution below works). Regardless, the code below uses this provider `container` to read and set the BLE and MQTT states depending on the current Geofence status.

```dart
Future<void> _onGeofenceStatusChanged(
    Geofence geofence,
    GeofenceRadius geofenceRadius,
    GeofenceStatus geofenceStatus,
    Location location) async {
	// some print statements
  _geofenceStreamController.sink.add(geofenceStatus);
  if (geofenceStatus == GeofenceStatus.ENTER) {
    container.read(clientStateProvider.notifier).connect();
    container.read(beaconStateProvider.notifier).bleOnSwitch();
  } else if (geofenceStatus == GeofenceStatus.EXIT) {
    container.read(clientStateProvider.notifier).disconnect();
    container.read(beaconStateProvider.notifier).bleKillSwitch();
  }
}
```

## Conclusion
I was able to leverage the awesome Riverpod library to make a basic yet functioning Flutter app that dynamically connects with the ESP32 and the Raspberry Pi through BLE and MQTT, respectively.

Having spent a bit of time now with Flutter and state management, I have learnt to just follow what Remi (the creator of Riverpod) says when it comes to Flutter - and this Riverpod library, while quite opinionated, is ultimately very easy to use once you get the hang of it.

## Flutter Tips
### How do you test a Geofence in an emulator?

All we need to be able to do is manipulate the device's GPS coordinates. Its as simple as selecting the three dot icon beside the emulator, then selecting Location. Here you can enter whatever coordinates you want (just go to Google Maps and find your home coordinates - can be found in the URL) and then hit Send.

<figure>
<img src="https://storage.googleapis.com/craigstanton-public-assets/images/smarthome/android_emulator_gps_coords.png"/>
	
</figure>

### How can you see the Lifecycle Events when testing?

It was extremely useful to print in the logs the Flutter lifecycle events when building this app. There is tonnes of documentation on how to set this up, however here is a brief overview for my own understanding.

First, the main Stateful Widget must add the mixin `WidgetsBindingObserver` and immediately add the observer:

```dart

class _MyHomePageState extends ConsumerState<MyHomePage>
    with WidgetsBindingObserver {
  @override
  void initState() {
    super.initState();
    geofenceCallbacks();
	// Add the observer
    WidgetsBinding.instance!.addObserver(this);
    permissions.requestPermission();
  }

  // ...
	
}
```

Once that is done, we can monitor lifecycle events by adding the following code. Note that we need to dispose as usual:

```dart
@override
  void dispose() {
    WidgetsBinding.instance!.removeObserver(this);
  	container.read(beaconStateProvider.notifier).bleKillSwitch();
  	super.dispose();
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    super.didChangeAppLifecycleState(state);
    print("App Lifecycle State: ${state}");
  }
```