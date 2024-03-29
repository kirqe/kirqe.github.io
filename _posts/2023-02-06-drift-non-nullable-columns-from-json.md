---
layout: post
title: Handling Drift non-nullable columns and data coming from JSON
category: posts
---

Using [Drift](https://pub.dev/packages/drift), with the latest version of Flutter, the ID column is not nullable by default(because of the null-safety)
We could make it nullable, but it's a primary key. 

```dart
  IntColumn get id => integer().autoIncrement()();
```

The data comes from json. `fromJson` returns the full row, and with null-safety, it's not possible to not read the ID. Which is null... But we need a source for the ID column.



With the help of the following package and [Drift custom row classes](https://drift.simonbinder.eu/docs/advanced-features/custom_row_classes/) we can omit the ID column and allow it to be autogenerated.

```
  json_annotation: ^4.8.0
  json_serializable: ^6.6.1
```

Most posts in the issues on Github related to this question suggest using `Companion.insert(body: body)`
In this case, we don't have to provide the ID value to insert a record.

But this approach doesn't allow us to use DAOs and things like `insertOnConflictUpdate`.

Let's say you have the following deserializer that accepts JSON and creates a list of post companions:

```dart
class PostsDeserializer {
  final List json;
  final Set<PostsCompanion> postsCompanions;

  PostsDeserializer({required this.json}) : postsCompanions = {} {
    _call();
  }

  void _call() {
    json.forEach((postsJson) {
      postsCompanions.add(
        Post.fromJson({
          'body': postsJson['body'],
          'published_at': postsJson['published_at']
        }).toCompanion(),
      );
    });
  }
}

```

It's then called with

```dart
final postsDeserializer = PostsDeserializer(json: postsRespose.data);

// and then saved with
Future _savePostsData(PostsDeserializer deserializer) async {
  deserializer.postsCompanions.forEach((companion) async {
    await db.postsDao.insertOrUpdate(companion);
  });
}

```

The model for the posts table may look like the following block:

```dart
import 'package:json_annotation/json_annotation.dart' as ja;

part 'posts.g.dart';

@UseRowClass(Post) // use the custom class bellow
class Posts extends Table {
  IntColumn get id => integer().autoIncrement()();
  
  TextColumn get body => text()();
  
  DateTimeColumn get publishedAt => dateTime()();
}

@ja.JsonSerializable()
class Post implements Insertable<Post> {
  final int? id;
  
  final String body;
  
  // the incoming json key has a different casing
  @ja.JsonKey(name: 'published_at')
  final String publishedAt;

  PersonService({this.id, required this.body});

  factory Post.fromJson(Map<String, dynamic> json) => _$PostFromJson(json);

  bool get isNew => id == null;

  Map<String, dynamic> toJson() => _$PostToJson(this);

  @override
  String toString() => toJson().toString();

  PostsCompanion toCompanion() {
    if (isNew) {
      // new record - we don't have the id
      return PostCompanion.insert(body: body, createdAt: createdAt);
    } else {
      return PersonServicesCompanion(
          id: Value(id ?? 0), 
          body: Value(body), 
          createdAt: Value(createdAt));
    }
  }

  // implements Insertable
  // https://drift.simonbinder.eu/docs/advanced-features/custom_row_classes/#inserts-and-updates-with-custom-classes
  @override 
  Map<String, Expression<Object>> toColumns(bool nullToAbsent) {
    return PostsCompanion(body: Value(body), createdAt: Value(createdAt))
            .toColumns(nullToAbsent);
  }

  Post copyWith({String? body}) => Post(body: body ?? this.body);
}

```

After running `flutter pub run build_runner build --delete-conflicting-outputs` the `posts.g.dart` is going to look like the following:

```dart
// GENERATED CODE - DO NOT MODIFY BY HAND

part of 'posts.dart';

// **************************************************************************
// JsonSerializableGenerator
// **************************************************************************

Post _$PostFromJson(Map<String, dynamic> json) => Post(
      id: json['id'] as int?,
      body: json['body'] as String,
      publishedAt: DateTime.parse(json['published_at'] as String),
    );
	
Map<String, dynamic> _$PostToJson(Post instance) => <String, dynamic>{
      'id': instance.id,
      'body': instance.body,
      'published_at': instance.publishedAt,
    };
	
```





