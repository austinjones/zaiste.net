+++
title = "Create a Singleton in Dart"
[taxonomies]
topics = [ "Dart" ]
+++

Dart provides factory constructors, which simplifies the creation of singletons.

```dart
class Singleton {
  static final Singleton _singleton = Singleton._internal();

  factory Singleton() => _singleton;

  Singleton._internal(); // private constructor
}

main() {
  var s1 = Singleton();
  var s2 = Singleton();

  print(identical(s1, s2));  // true
  print(s1 == s2);           // true
}
```

The `factory` construct specifies that whenever there is a request for a new `Singleton` instance, run the specified constructor function.

Another way is to use a static field that use a private constructor. This way it is not possible to create a new instance using `Singleton()`, the reference to the only instance is available only through the `instance` static field.

```dart
class Singleton {
  Singleton._privateConstructor();

  static final Singleton instance = Singleton._privateConstructor();
}

main() {
  var s1 = Singleton.instance;
  var s2 = Singleton.instance;

  print(identical(s1, s2));  // true
  print(s1 == s2);           // true
}
```

A singleton can be used for a Session storage

```dart
class Session {
  // singleton
  static final Session _singleton = Session._internal();
  factory Session() => _singleton;
  Session._internal();
  static Session get shared => _singleton;

  String username;
  String password;
}

Session.shared.username = 'abc';
```

