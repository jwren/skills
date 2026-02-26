---
name: "architecture"
description: "Configure Flutter SDK on the user's machine, setup IDEs, and diagnose CLI errors."
metadata:
  urls:
    - "https://docs.flutter.dev/app-architecture"
    - "https://docs.flutter.dev/app-architecture/concepts"
    - "https://docs.flutter.dev/app-architecture/guide"
    - "https://docs.flutter.dev/resources/architectural-overview"
    - "https://docs.flutter.dev/app-architecture/recommendations"
    - "https://docs.flutter.dev/app-architecture/design-patterns"
    - "https://docs.flutter.dev/app-architecture/case-study"
    - "https://docs.flutter.dev/app-architecture/case-study/data-layer"
    - "https://docs.flutter.dev/app-architecture/design-patterns/key-value-data"
    - "https://docs.flutter.dev/learn/pathway/how-flutter-works"
  model: "models/gemini-3.1-pro-preview"
  last_modified: "Thu, 26 Feb 2026 22:58:46 GMT"

---
# Flutter App Architecture Implementation

Analyzes application requirements and implements a scalable, maintainable Flutter architecture using the MVVM pattern. Enforces strict separation of concerns across UI, Domain, and Data layers, utilizes unidirectional data flow, and manages state via `ChangeNotifier` and `ListenableBuilder`.

## Goal
Implement a robust Flutter feature following the official layered architecture guidelines (MVVM). The end state is a fully wired feature comprising Services, Repositories, ViewModels, and Views, with proper dependency injection and error handling. It assumes a standard Flutter environment using `ChangeNotifier` for state management and a dependency injection mechanism (like `provider`).

## Decision Logic

When scaffolding a new feature, use the following decision tree to determine the required architectural components:

1. **Data Source Complexity:**
   * *If* the feature requires fetching data from an external API or local database -> Create a **Service** class.
   * *If* the feature only relies on existing in-memory state -> Skip Service creation.
2. **Data Management & SSOT (Single Source of Truth):**
   * *If* the feature reads/writes data -> Create a **Repository** to transform raw Service data into Domain Models.
   * *If* multiple environments (e.g., local vs. remote) are needed -> Create an abstract `Repository` interface and concrete implementations.
3. **Business Logic Complexity:**
   * *If* the feature requires merging data from multiple repositories or has highly complex, reusable logic -> Create a **Domain Layer (Use-cases)**.
   * *If* the logic is straightforward -> Skip the Domain Layer and allow the ViewModel to interact directly with the Repository.

## Instructions

1. **Analyze Feature Scope and Architecture Requirements**
   Evaluate the requested feature against the Decision Logic above. 
   **STOP AND ASK THE USER:** "Does this feature require complex business logic or merging multiple data sources that warrants a dedicated Domain Layer (Use-cases)? Should I define an abstract Repository interface for multiple environments (e.g., local/mock vs. remote)?"

2. **Define Domain Models and Result Wrappers**
   Implement immutable domain models and a `Result` wrapper for safe error handling across layers.
   ```dart
   // result.dart
   sealed class Result<T> {}
   class Ok<T> extends Result<T> {
     final T value;
     Ok(this.value);
   }
   class Error<T> extends Result<T> {
     final Exception exception;
     Error(this.exception);
   }

   // user_model.dart
   import 'package:flutter/foundation.dart';

   @immutable
   class User {
     final String id;
     final String name;
     const User({required this.id, required this.name});
   }
   ```

3. **Implement the Data Layer (Services)**
   Create stateless service classes to wrap external APIs. Services must only return raw data or `Result` objects and hold no state.
   ```dart
   // api_service.dart
   class ApiService {
     Future<Result<Map<String, dynamic>>> fetchUserData(String userId) async {
       try {
         // Simulate network call
         final rawData = {'id': userId, 'name': 'Jane Doe'};
         return Ok(rawData);
       } on Exception catch (e) {
         return Error(e);
       }
     }
   }
   ```

4. **Implement the Data Layer (Repositories)**
   Create the repository to act as the Single Source of Truth. It must consume the Service, handle caching/retries, and output Domain Models.
   ```dart
   // user_repository.dart
   import 'dart:async';

   class UserRepository {
     final ApiService _apiService;
     final _userController = StreamController<User?>.broadcast();
     User? _cachedUser;

     UserRepository(this._apiService);

     Stream<User?> observeUser() => _userController.stream;

     Future<Result<void>> loadUser(String userId) async {
       final result = await _apiService.fetchUserData(userId);
       if (result is Ok<Map<String, dynamic>>) {
         _cachedUser = User(
           id: result.value['id'], 
           name: result.value['name']
         );
         _userController.add(_cachedUser);
         return Ok(null);
       } else if (result is Error<Map<String, dynamic>>) {
         return Error(result.exception);
       }
       return Error(Exception('Unknown error'));
     }
   }
   ```

5. **Implement the UI Layer (ViewModel)**
   Create a `ChangeNotifier` to manage UI state. It must consume the Repository (or Use-cases), expose data for the View, and implement Commands for user actions.
   ```dart
   // user_view_model.dart
   import 'package:flutter/foundation.dart';
   import 'dart:async';

   class UserViewModel extends ChangeNotifier {
     final UserRepository _userRepository;
     StreamSubscription<User?>? _subscription;

     User? _user;
     User? get user => _user;

     bool _isLoading = false;
     bool get isLoading => _isLoading;

     UserViewModel(this._userRepository) {
       _subscription = _userRepository.observeUser().listen((newUser) {
         _user = newUser;
         notifyListeners();
       });
     }

     Future<void> fetchUser(String userId) async {
       _isLoading = true;
       notifyListeners();

       final result = await _userRepository.loadUser(userId);
       if (result is Error) {
         // Handle error state
       }

       _isLoading = false;
       notifyListeners();
     }

     @override
     void dispose() {
       _subscription?.cancel();
       super.dispose();
     }
   }
   ```

6. **Implement the UI Layer (View)**
   Create the UI using `ListenableBuilder` to react to ViewModel changes. Ensure absolutely no business logic exists in this file.
   ```dart
   // user_view.dart
   import 'package:flutter/material.dart';

   class UserView extends StatelessWidget {
     final UserViewModel viewModel;

     const UserView({Key? key, required this.viewModel}) : super(key: key);

     @override
     Widget build(BuildContext context) {
       return Scaffold(
         appBar: AppBar(title: const Text('User Profile')),
         body: ListenableBuilder(
           listenable: viewModel,
           builder: (context, _) {
             if (viewModel.isLoading) {
               return const Center(child: CircularProgressIndicator());
             }
             final user = viewModel.user;
             if (user == null) {
               return Center(
                 child: ElevatedButton(
                   onPressed: () => viewModel.fetchUser('123'),
                   child: const Text('Load User'),
                 ),
               );
             }
             return Center(child: Text('Hello, ${user.name}'));
           },
         ),
       );
     }
   }
   ```

7. **Validate and Fix**
   Review the generated implementation against the constraints. 
   * Verify that data flows unidirectionally (Service -> Repository -> ViewModel -> View).
   * Verify that the View contains no logic other than layout and routing.
   * If any business logic is found in the View, refactor it immediately into the ViewModel.

## Constraints

* **Strict Layer Isolation:** Views must NEVER directly access Services or Repositories. They may only interact with ViewModels.
* **No UI Logic in Views:** Views must only contain layout logic, simple if-statements for rendering, and routing. All data transformations and state mutations must occur in the ViewModel.
* **Single Source of Truth:** Repositories must be the only classes that mutate application data. ViewModels must read from Repositories.
* **Immutable Models:** All domain models must be immutable. Use `const` constructors and `final` fields.
* **Dependency Injection:** Services must be injected into Repositories, and Repositories must be injected into ViewModels. Do not instantiate these dependencies directly inside the classes.
