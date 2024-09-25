# Dart and Flutter Style Guide: Clean Architecture and Bloc Pattern

## Introduction

This style guide is meant to help developers understand how to structure and organize their code when using **Dart** and **Flutter**, specifically following the **Bloc Pattern** for state management and **Clean Architecture**. These guidelines will help you produce consistent, scalable, and maintainable code.

The Bloc pattern is particularly powerful because it helps separate business logic from UI. Clean Architecture further enhances the separation of concerns by dividing the app into **presentation**, **domain**, and **data** layers.

---

## 1. Project Structure

### Why a Structured Approach?
When developing larger applications, having a well-defined folder structure is essential for:
- **Scalability**: Easily extend the app by adding new features or services.
- **Maintainability**: Find and update code without introducing bugs.
- **Separation of Concerns**: Keep UI logic, business logic, and data operations isolated from one another.

### Example Project Structure
We will follow the **Clean Architecture** principle to divide the project into three core layers: `data`, `domain`, and `presentation`.

```
lib/
├── core/
│   ├── error/
│   ├── usecases/
│   ├── utils/
├── features/
│   └── feature_name/
│       ├── data/
│       │   ├── models/
│       │   ├── datasources/
│       │   └── repositories/
│       ├── domain/
│       │   ├── entities/
│       │   ├── repositories/
│       │   └── usecases/
│       └── presentation/
│           ├── blocs/
│           └── pages/
├── injection_container.dart
└── main.dart
```

### Core Components of Structure
- **core/**: Contains utilities or services that are shared across features, such as global error handling, input validation, or reusable entities like `Failure` classes.
  
- **features/**: A folder for each feature/module. Each feature contains its own `data`, `domain`, and `presentation` layers, making the app modular and scalable. For instance, you might have a `login/` feature or a `user_profile/` feature.

- **injection_container.dart**: Used to handle dependency injection. It registers classes and services for injection across the app, using a service locator such as `GetIt`.

---

## 2. Naming Conventions

### Why Naming Conventions Matter
Consistent naming conventions make the code readable and self-documenting. New team members can easily understand what each component is responsible for. Follow these naming conventions across all your projects.

### Naming Rules
- **Classes**: Use `PascalCase`. Each word in the name should start with an uppercase letter. Example: `UserRepository`, `LoginUseCase`.
  
- **Methods and Variables**: Use `camelCase`. The first word is lowercase, and each subsequent word starts with an uppercase letter. Example: `fetchUserData`, `isLoading`.

- **File Names**: Use `snake_case`. All characters are lowercase, and words are separated by underscores. Example: `login_page.dart`, `user_repository.dart`.

### Bloc Naming Convention
- **Blocs**: Name your Bloc class based on the feature it handles, followed by the suffix `Bloc`. 
Example: 
```
class LoginBloc extends Bloc<LoginEvent, LoginState> {
  ...
}
```
- **States and Events**: Create separate files for each state and event. State and event names should clearly indicate the action or condition.
Example: 
```
class UserLoading extends UserState {}
```

---

## 3. Clean Architecture Layers in Detail

Clean Architecture divides your application into three key layers, each with a specific responsibility.

### 3.1. Data Layer
The data layer handles communication with external sources (APIs, databases, etc.). It contains models, data sources (local or remote), and repositories. 

**Repositories** act as an abstraction between the data layer and domain layer.

#### Example of Remote Data Source

```
abstract class UserRemoteDataSource {
  Future<UserModel> getUser();
}

class UserRemoteDataSourceImpl implements UserRemoteDataSource {
  final http.Client client;
  
  UserRemoteDataSourceImpl(this.client);

  @override
  Future<UserModel> getUser() async {
    final response = await client.get(Uri.parse('https://api.example.com/user'));
    return UserModel.fromJson(json.decode(response.body));
  }
}
```

In this example, the `UserRemoteDataSourceImpl` class communicates with an external API to fetch user data.

#### Best Practice:
- Handle all API responses properly, using HTTP status codes to detect errors.
- Use repositories to abstract data sources (remote or local) to decouple the business logic from data-fetching concerns.

---

### 3.2. Domain Layer
The **domain layer** is the most critical part of the app because it holds the **business logic**. This layer consists of:
- **Entities**: Core data objects. They are plain Dart classes without any dependencies on Flutter or any external data sources.
- **Use Cases**: Business rules. They contain application-specific logic.

#### Example of an Entity
```
class User {
  final String name;
  final int age;

  User({required this.name, required this.age});
}
```

#### Example of a Use Case
```
class GetUser {
  final UserRepository repository;

  GetUser(this.repository);

  Future<Either<Failure, User>> call() {
    return repository.getUser();
  }
}
```

The `GetUser` use case fetches user data by calling the repository. The use of `Either` ensures that the operation can return either a success or a failure.

#### Best Practice:
- Keep the domain layer **pure** by ensuring it doesn't depend on any Flutter packages or APIs.
- Use cases should be focused on a single operation, keeping your code easy to understand and maintain.

---

### 3.3. Presentation Layer
The **presentation layer** handles everything related to the UI. This is where you will use the **Bloc** pattern to manage state. The UI layer interacts with the Bloc to get the current state and respond to user actions.

#### Bloc Class Example

```
class UserBloc extends Bloc<UserEvent, UserState> {
  final GetUser getUser;

  UserBloc({required this.getUser}) : super(UserInitial());

  @override
  Stream<UserState> mapEventToState(UserEvent event) async* {
    if (event is GetUserEvent) {
      yield UserLoading();
      final failureOrUser = await getUser();
      yield failureOrUser.fold(
        (failure) => UserError(message: 'Failed to load user'),
        (user) => UserLoaded(user: user),
      );
    }
  }
}
```

#### BlocBuilder Example
Use `BlocBuilder` to respond to changes in the Bloc's state.

```
class UserPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (_) => UserBloc(getUser: GetUser(UserRepositoryImpl())),
      child: Scaffold(
        appBar: AppBar(title: Text('User Page')),
        body: BlocBuilder<UserBloc, UserState>(
          builder: (context, state) {
            if (state is UserLoading) {
              return CircularProgressIndicator();
            } else if (state is UserLoaded) {
              return Text('Hello, ${state.user.name}');
            } else if (state is UserError) {
              return Text('Error: ${state.message}');
            }
            return Container();
          },
        ),
      ),
    );
  }
}
```

#### Best Practice:
- **Avoid putting business logic in the UI**. The UI should only display information and trigger events, leaving all business logic to Blocs and Use Cases.
- **Divide responsibilities in the Bloc**: Events trigger state transitions, and the state represents what the UI should display.

---

## 4. Dependency Injection

Dependency Injection (DI) decouples object creation from object use, which promotes flexibility and testability. In Flutter, we often use **GetIt** for dependency injection.

### Example: Setting Up GetIt

To set up GetIt, register your dependencies in `injection_container.dart`:

```
final sl = GetIt.instance;

void init() {
  // Bloc
  sl.registerFactory(() => UserBloc(getUser: sl()));

  // Use cases
  sl.registerLazySingleton(() => GetUser(sl()));

  // Repositories
  sl.registerLazySingleton<UserRepository>(
    () => UserRepositoryImpl(remoteDataSource: sl(), localDataSource: sl())
  );

  // Data sources
  sl.registerLazySingleton<UserRemoteDataSource>(
    () => UserRemoteDataSourceImpl(client: sl())
  );

  // External dependencies
  sl.registerLazySingleton(() => http.Client());
}
```

### Best Practice:
- Ensure all external services, such as APIs or database clients, are injected via GetIt.
- Use `registerLazySingleton` for services that should persist throughout the app’s lifecycle and `registerFactory` for Blocs that are created and disposed of frequently.

---

## 5. Error Handling

Consistent error handling improves user experience and ensures that the application is robust against unexpected failures.

### Failure Model
Define a **Failure** class to represent errors throughout the app:

```
class Failure {
  final String message;

  Failure(this message);
}
```

This `Failure` class can be used across the application to represent error states in a consistent way, typically in the domain and data layers.

### Example: Handling Failures in Use Cases

In a use case, you can handle errors using the `Either` type from functional programming. This ensures that the use case can return either a success or a failure.

```
class GetUser {
  final UserRepository repository;

  GetUser(this.repository);

  Future<Either<Failure, User>> call() async {
    try {
      final user = await repository.getUser();
      return Right(user);
    } catch (e) {
      return Left(Failure('Failed to fetch user'));
    }
  }
}
```

This approach makes it easy to handle both success and failure cases in the Bloc and presentation layer.

---

## 6. Bloc Best Practices

### Event-Driven State Management
Blocs are designed to react to **events**. Each event corresponds to a user action or application lifecycle change, which triggers the corresponding **state transition**.

#### Specific Events and States
Define specific events and states to capture every possible state of your feature. For example, in a user management feature, you may have the following events and states:

#### User Events:
```
abstract class UserEvent {}

class GetUserEvent extends UserEvent {}
class UpdateUserEvent extends UserEvent {
  final User user;

  UpdateUserEvent({required this.user});
}
```

#### User States:
```
abstract class UserState {}

class UserInitial extends UserState {}
class UserLoading extends UserState {}
class UserLoaded extends UserState {
  final User user;

  UserLoaded({required this.user});
}
class UserError extends UserState {
  final String message;

  UserError({required this.message});
}
```

### Yielding States
Each Bloc event triggers a series of state transitions. For example, a `GetUserEvent` might transition through `UserLoading` and `UserLoaded` or `UserError`, depending on whether the operation succeeds or fails.

```
class UserBloc extends Bloc<UserEvent, UserState> {
  final GetUser getUser;

  UserBloc({required this.getUser}) : super(UserInitial());

  @override
  Stream<UserState> mapEventToState(UserEvent event) async* {
    if (event is GetUserEvent) {
      yield UserLoading();
      final failureOrUser = await getUser();
      yield failureOrUser.fold(
        (failure) => UserError(message: failure.message),
        (user) => UserLoaded(user: user),
      );
    }
  }
}
```

#### Best Practice:
- Always use descriptive state names that reflect the current condition of the UI or data.
- Handle all possible states (like loading, error, and loaded) in the Bloc and represent them in the UI.

---

## 7. Testing

### Why Testing is Critical
Testing ensures the reliability and correctness of your code. With Clean Architecture, each layer is isolated, making it easier to test components individually.

#### Types of Testing:
- **Unit Tests**: Test individual methods or functions, like use cases.
- **Widget Tests**: Verify the UI behavior and interactions between components.
- **Integration Tests**: Ensure that all components work together in a real-world environment.

### Testing a Use Case
Use mock repositories to test use cases. Here’s an example of how to test the `GetUser` use case.

```
class MockUserRepository extends Mock implements UserRepository {}

void main() {
  late GetUser getUser;
  late MockUserRepository mockUserRepository;

  setUp(() {
    mockUserRepository = MockUserRepository();
    getUser = GetUser(mockUserRepository);
  });

  test('should return user when repository fetches data successfully', () async {
    // Arrange
    final user = User(name: 'John Doe', age: 30);
    when(mockUserRepository.getUser()).thenAnswer((_) async => Right(user));

    // Act
    final result = await getUser();

    // Assert
    expect(result, Right(user));
  });

  test('should return Failure when repository fails to fetch data', () async {
    // Arrange
    when(mockUserRepository.getUser()).thenAnswer((_) async => Left(Failure('Error fetching data')));

    // Act
    final result = await getUser();

    // Assert
    expect(result, Left(Failure('Error fetching data')));
  });
}
```

#### Bloc Testing
You can test Blocs by simulating events and verifying that they emit the correct sequence of states.

```
void main() {
  late UserBloc userBloc;
  late MockGetUser mockGetUser;

  setUp(() {
    mockGetUser = MockGetUser();
    userBloc = UserBloc(getUser: mockGetUser);
  });

  test('should emit [UserLoading, UserLoaded] when data is fetched successfully', () async {
    // Arrange
    final user = User(name: 'John Doe', age: 30);
    when(mockGetUser()).thenAnswer((_) async => Right(user));

    // Assert later
    final expected = [UserLoading(), UserLoaded(user: user)];
    expectLater(userBloc.stream, emitsInOrder(expected));

    // Act
    userBloc.add(GetUserEvent());
  });
}
```

---

## 8. Flutter UI Best Practices

### Stateless vs Stateful Widgets
- **Stateless Widgets**: Use these when your widget doesn’t need to hold state. Stateless widgets are more performant because they are immutable.
  
- **Stateful Widgets**: Use these when your widget needs to update based on user interaction or data changes. However, avoid placing business logic in Stateful widgets; instead, delegate that to Blocs or other state management solutions.

#### Example:
Use Stateless Widgets with Bloc for a clean separation of logic and UI.

```
class UserPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (_) => UserBloc(getUser: GetUser(UserRepositoryImpl())),
      child: Scaffold(
        appBar: AppBar(title: Text('User Page')),
        body: BlocBuilder<UserBloc, UserState>(
          builder: (context, state) {
            if (state is UserLoading) {
              return Center(child: CircularProgressIndicator());
            } else if (state is UserLoaded) {
              return Text('Hello, ${state.user.name}');
            } else if (state is UserError) {
              return Text('Error: ${state.message}');
            }
            return Container();
          },
        ),
      ),
    );
  }
}
```

---

## 9. Writing Documentation Inside Code

### Why Documenting Code Matters

Effective code documentation is critical in creating maintainable and readable code, especially when working in teams or when handing over projects to new developers. Documentation allows developers to quickly understand the purpose and behavior of classes, methods, and variables.

In Dart and Flutter, good documentation:
- **Improves readability**: Clear comments make code easier to follow.
- **Reduces misunderstandings**: Well-documented code helps prevent incorrect usage of methods or classes.
- **Acts as a guide**: When another developer revisits the code after some time, documentation serves as a helpful reference.
- **Helps with auto-generated documentation**: Tools like `dartdoc` can generate comprehensive documentation directly from code comments.

### 9.1. Documentation Comments (`///`)

In Dart, use the triple-slash `///` syntax to add documentation comments above classes, methods, and properties. These comments provide descriptions that tools like `dartdoc` can extract and convert into helpful documentation.

#### Example:
```
/// A use case class responsible for fetching user details.
///
/// The [GetUser] use case interacts with the [UserRepository] to fetch user data.
/// It returns either a [User] on success or a [Failure] on error.
class GetUser {
  final UserRepository repository;

  GetUser(this.repository);

  /// Executes the use case to fetch a [User].
  ///
  /// This method interacts with the repository and attempts to fetch
  /// the user data. It returns an [Either] containing a [Failure] or a [User].
  /// 
  /// Example:
  /// ```
  /// final result = await getUser();
  /// result.fold(
  ///   (failure) => handleError(failure),
  ///   (user) => handleSuccess(user),
  /// );
  /// ```
  Future<Either<Failure, User>> call() {
    return repository.getUser();
  }
}
```

### 9.2. Best Practices for Writing Documentation Comments

1. **Be Concise but Informative**: Documentation should explain what the code does in a clear and straightforward way without overwhelming the reader with too much detail.

   **Bad:**
   ```
   /// This method is used to get the user and it returns either a failure or user
   ```
   
   **Good:**
   ```
   /// Fetches a [User] from the repository.
   ///
   /// Returns [Failure] if an error occurs, otherwise it returns a [User].
   ```

2. **Explain Parameters and Return Types**: Clearly describe the purpose of function parameters and return values, especially if they are complex types.

   **Example:**
   ```
   /// Retrieves a user by ID from the repository.
   ///
   /// [id] is the unique identifier of the user to be fetched.
   /// Returns a [Future] that completes with either a [Failure] or a [User].
   Future<Either<Failure, User>> getUserById(String id);
   ```

3. **Use Proper Formatting**: 
   - Use markdown within the documentation comments to highlight code snippets, examples, or references.
   - Provide **code examples** inside comments to illustrate how a method or class should be used.

   **Example:**
   ```
   /// Returns a user after fetching from a remote data source.
   ///
   /// Example:
   /// ```
   /// final user = await getUser();
   /// user.fold(
   ///   (failure) => print('Error: \$failure'),
   ///   (user) => print('User: \$user'),
   /// );
   /// ```
   Future<Either<Failure, User>> getUser();
   ```

4. **Document Edge Cases and Behavior**: Describe edge cases or important details that developers should be aware of when using the method or class.

   **Example:**
   ```
   /// Deletes a user from the system.
   ///
   /// **Note:** If the user does not exist, this method will throw a [UserNotFoundException].
   Future<void> deleteUser(String id);
   ```

5. **Update Comments Regularly**: Make sure that your comments stay up-to-date with the code. If a function or class changes, ensure the documentation reflects these changes.

6. **Use TODO Comments**: If a feature is incomplete or a method needs to be optimized, use `// TODO` comments. This helps developers recognize areas that need improvement.

   **Example:**
   ```
   // TODO: Optimize this method to handle large data sets.
   Future<List<User>> fetchAllUsers() {
     // Current implementation
   }
   ```

### 9.3. In-line Comments (`//`)

In-line comments provide short explanations for small chunks of code that may not be immediately understandable, such as complex business logic or non-obvious decisions. These comments should be used sparingly to avoid cluttering the code.

#### Example:
```
// We use a 2-second delay here to simulate a network call.
await Future.delayed(Duration(seconds: 2));
```

#### Best Practices:
1. **Use for Clarification**: Only use in-line comments to explain **why** something is done in a particular way or to explain complex code that might not be immediately understandable.

2. **Avoid Stating the Obvious**: Don’t use comments to describe code that’s self-explanatory. For example, comments like `// Get the user` right above a call to `getUser()` are unnecessary.

   **Bad:**
   ```
   // Get the user from the database
   final user = await getUser();
   ```

   **Good:**
   ```
   // Retry fetching user in case of a temporary network error.
   final user = await retry(getUser, retries: 3);
   ```

---

## 10. Consistent Formatting and Code Readability

While writing code, maintaining a consistent structure and formatting style is crucial for readability. The following guidelines help in ensuring that your code is clean, consistent, and easy to read.

### 10.1. Line Length
- Limit each line of code to **80-100 characters**. This keeps code readable, especially in side-by-side diff views during code reviews.
  
  **Example:**
  ```
  // Bad: Too long, unreadable on small screens
  final user = await repository.fetchUserFromApiAndStoreInDatabaseWithUniqueIdentifier(userId, apiToken);

  // Good: Wrapped across multiple lines
  final user = await repository
    .fetchUserFromApiAndStoreInDatabaseWithUniqueIdentifier(
      userId, apiToken
    );
  ```

### 10.2. Indentation
- Use **two spaces** per indentation level. Avoid using tabs for indentation.

### 10.3. Blank Lines
- Add blank lines between logical blocks of code (such as between variable declarations and method calls) to enhance readability.

  **Example:**
  ```
  final user = await getUser();

  if (user != null) {
    print('User found');
  } else {
    print('User not found');
  }
  ```

---

## Conclusion

This guide covers the key aspects of writing clean, maintainable, and scalable applications using **Dart** and **Flutter**, following **Bloc** and **Clean Architecture** principles. These practices will help you and your team build applications that are easier to maintain and extend over time, ensuring that all code is organized, well-tested, and robust.

By following this guide, you’ll be able to:
1. Structure your project for scalability.
2. Implement business logic in a maintainable way using Clean Architecture.
3. Manage state effectively with the Bloc pattern.
4. Write testable code by isolating concerns.
5. Ensure consistent error handling and clear documentation.

Welcome to the world of professional Flutter development, and happy coding!
