---
title: Flutter concurrency for Swift developers
description: Leverage your Swift concurrency knowledge while learning Flutter and Dart
---

<?code-excerpt path-base="resources"?>

Both Dart and Swift support concurrent programming. 
This guide should help you understand how
concurrency works in Dart and how it compares to Swift.
With this understanding, you can create
high-performing iOS apps. 

When developing in the Apple ecosystem, 
some tasks might take a long time to complete. 
These tasks include fetching or processing large amounts of data.
iOS developers typically use Grand Central Dispatch (GCD)
to schedule tasks using a shared thread pool.
With GCD, developers add tasks to dispatch queues
and GCD decides on which thread to execute them.

But, GCD spins up threads to 
handle remaining work items.
This means you can end up with a large number of threads 
and the system can become over committed.
With Swift, the structured concurrency model reduced the number 
of threads and context switches. 
Now, each core has only one thread.

Dart has a single-threaded execution model, 
with support for `Isolates`, an event loop, and asynchronous code. 
An `Isolate` is Dart's implementation of a lightweight thread.
Unless you spawn an `Isolate`, your Dart code runs in the 
main UI thread driven by an event loop. 
Flutter’s event loop is 
equivalent to the iOS main loop—in other words, 
the Looper attached to the main thread.

Dart’s single-threaded model doesn’t mean 
you are required to run everything 
as a blocking operation that causes the UI to freeze. 
Instead, use the asynchronous 
features that the Dart language provides, 
such as `async`/`await`.

### Asynchronous Programming
An asynchronous operation allows other operations 
to execute before it completes. 
Both Dart and Swift support asynchronous functions 
using the `async` and `await` keywords. 
In both cases, `async` marks that a function 
performs asynchronous work, 
and `await` tells the system to await a result 
from function. This means that the Dart VM _could_ 
suspend the function, if necessary. 
For more details on asynchronous programming, 
see [Concurrency in Dart]({{site.dart-site}}/guides/language/concurrency).

### Leveraging the main thread/isolate

For Apple operating systems, the primary (also called the main) 
thread is where the application begins running. 
Rendering the user interface always happens on the main thread. 
One difference between Swift and Dart is that  
Swift might use different threads for different tasks, 
and Swift doesn’t guarantee which thread is used. 
So, when dispatching UI updates in Swift, 
you might need to ensure that the work occurs on the main thread. 

Say you want to write a function that fetches the 
weather asynchronously and 
displays the results. 

In GCD, to manually dispatch a process to the main thread, 
you might do something like the following.  

First, define the `Weather` `enum`:

```swift
// 1 second delay is used in mocked API call. 
extension UInt64 {
  static let oneSecond = UInt64(1_000_000_000)
} 

enum Weather: String {
    case rainy, sunny
}
```

Next, define the view model and mark it as an `ObservableObject` 
so that it can return a value of type `Weather?`. 
Use GCD create to a `DispatchQueue` to 
send the work to the pool of threads

```swift
class ContentViewModel: ObservableObject {
    @Published private(set) var result: Weather?

    private let queue = DispatchQueue(label: "weather_io_queue")
    func load() {
        // Mimic 1 sec delay.
        queue.asyncAfter(deadline: .now() + 1) { [weak self] in
            DispatchQueue.main.async {
                self?.result = .sunny
            }
        }
    }
}
```

Finally, display the results:

```swift
struct ContentView: View {
    @StateObject var viewModel = ContentViewModel()
    var body: some View {
        Text(viewModel.result?.rawValue ?? "Loading")
            .onAppear {
                viewModel.load()
        }
    }
}
```

More recently, Swift introduced _actors_ to support 
synchronization for shared, mutable state. 
To ensure that work is performed on the main thread,
define a view model class that is marked as a `@MainActor`, 
with a `load()` function that internally calls an 
asynchronous function using `Task`.   

```swift
@MainActor class ContentViewModel: ObservableObject {
  @Published private(set) var result: Weather?
  
  func load() async {
    try? await Task.sleep(nanoseconds: .oneSecond)
    self.result = .sunny
  }
}
```

Next, define the view model as a state object using `@StateObject`, 
with a `load()` function that can be called by the view model:

```swift
struct ContentView: View {
  @StateObject var viewModel = ContentViewModel()
  var body: some View {
    Text(viewModel.result?.rawValue ?? "Loading...")
      .task {
        await viewModel.load()
      }
  }
}
```

In Dart, all work runs on the main isolate by default. 
To implement the same example in Dart, 
first, create the `Weather` `enum`:

<?code-excerpt "lib/async_weather.dart (Weather)"?>
```dart
enum Weather {
  rainy,
  windy,
  sunny,
}
```

Then, define a simple view model (similar to what was created in SwiftUI), 
to fetch the weather. In Dart, a `Future` object represents a value to be
provided in the future. A `Future` is similar to Swift's `ObservableObject`. 
In this example, a function within the view model returns a `Future<Weather>` object:

<?code-excerpt "lib/async_weather.dart (HomePageViewModel)"?>
```dart
@immutable
class HomePageViewModel {
  const HomePageViewModel();
  Future<Weather> load() async {
    await Future.delayed(const Duration(seconds: 1));
    return Weather.sunny;
  }
}
```

The `load()` function in this example shares 
similarities with the Swift code. 
The Dart function is marked as `async` because
it uses the `await` keyword.

Additionally, a Dart function marked as `async`
automatically returns a `Future`.
In other words, you don't have to create a 
`Future` instance manually 
inside functions marked as `async`.

For the last step, display the weather value. 
In Flutter, [`FutureBuilder`]({{site.api}}/flutter/widgets/FutureBuilder-class.html) and 
[`StreamBuilder`]({{site.api}}/flutter/widgets/StreamBuilder-class.html)  
widgets are used to display the results of a Future in the UI. 
The following example uses a `FutureBuilder`:

<?code-excerpt "lib/async_weather.dart (HomePageWidget)"?>
```dart
class HomePage extends StatelessWidget {
  final HomePageViewModel viewModel = const HomePageViewModel();
  const HomePage({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return CupertinoPageScaffold(
      // Feed a FutureBuilder to your widget tree.
      child: FutureBuilder<Weather>(
        // Specify the Future that you want to track.
        future: viewModel.load(),
        builder: (context, snapshot) {
          // A snapshot is of type `AsyncSnapshot` and contains the
          // state of the Future. By looking if the snapshot contains
          // an error or if the data is null, you can decide what to
          // show to the user.
          if (snapshot.hasData) {
            return Center(
              child: Text(
                snapshot.data.toString(),
              ),
            );
          } else {
            return const Center(
              child: CupertinoActivityIndicator(),
            );
          }
        },
      ),
    );
  }
}
```

{% comment %}
Add the following text back when the example is available.
For the complete example, see the [async_weather][] file on GitHub. 
[async_weather]: {{site.github}}/flutter/website/tree/main/examples/resources/lib/async_weather.dart
{% endcomment %}

### Leveraging a background thread/isolate

Flutter apps can run on a variety of multi-core hardware, 
including devices running macOS and iOS. 
To improve the performance of these applications, 
you must sometimes run tasks on different cores
concurrently. This is especially important 
to avoid blocking UI rendering with long-running operations. 

In Swift, you can leverage GCD to run tasks on global queues
with different quality of service class (qos) properties. 
This indicates the task's priority.

```swift
func parse(string: String, completion: @escaping ([String:Any]) -> Void) {
  // Mimic 1 sec delay.
  DispatchQueue(label: "data_processing_queue", qos: .userInitiated)
    .asyncAfter(deadline: .now() + 1) {
      let result: [String:Any] = ["foo": 123]
      completion(result)
    }
  }
}
```

In Dart, you can offload computation to a worker isolate, 
often called a background worker. 
A common scenario spawns a simple worker isolate and 
returns the results in a message when the worker exits. 
As of Dart 2.19, you can use `Isolate.run()` to 
spawn an isolate and run computations:

```dart
void main() async {
  // Read some data.
  final jsonData = await Isolate.run(() => jsonDecode(jsonString) as Map<String, dynamic>);`

  // Use that data.
  print('Number of JSON keys: ${jsonData.length}');
}
```

In Flutter, you can also use the `compute` function 
to spin up an isolate to run a callback function:

```dart
final jsonData = await compute(getNumberOfKeys, jsonString);
```

In this case, the callback function is a top-level
function as shown below:

```dart
Map<String, dynamic> getNumberOfKeys(String jsonString) {
 return jsonDecode(jsonString);
}
```


You can find more information on Dart at
[Learning Dart as a Swift developer][], 
and more information on Flutter at
[Flutter for SwiftUI developers][] or [Flutter for UIKit developers][].

[Learning Dart as a Swift developer]: {{site.dart-site}}/guides/language/coming-from/swift-to-dart
[Flutter for SwiftUI developers]: {{site.url}}/get-started/flutter-for/swiftui-devs
[Flutter for UIKit developers]: {{site.url}}/get-started/flutter-for/uikit-devs
