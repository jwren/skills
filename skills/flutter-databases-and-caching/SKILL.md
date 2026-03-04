---
name: "flutter-databases-and-caching"
description: "Work with databases and cache data"
metadata:
  model: "models/gemini-3.1-pro-preview"
  last_modified: "Wed, 04 Mar 2026 18:13:34 GMT"

---
# Flutter SQLite and Caching Implementation

## Goal
Implements local persistence and caching strategies in Flutter applications using the `sqflite` and `path` packages. Handles database initialization, table creation, CRUD operations, and persistent caching logic to optimize I/O-bound network calls and ensure offline data availability.

## Decision Logic
Before implementing persistence, determine the appropriate caching and storage mechanism based on the data profile:
*   **Is the data temporary (single user session)?** Implement an in-memory cache (e.g., Dart `Map` or `List` within a Singleton or Provider).
*   **Is the data small and simple (e.g., user preferences, theme settings)?** Use `shared_preferences`.
*   **Is the data large, relational, or requires complex querying?** Use `sqflite` (Persistent Database Cache).

## Instructions

1. **Add Dependencies**
   Execute the following command to add the required packages to the Flutter project:
   ```bash
   flutter pub add sqflite path
   ```

2. **Define the Data Model**
   Create a data model class with `toMap()` and `fromMap()` methods to facilitate SQLite serialization.
   ```dart
   class CacheModel {
     final int id;
     final String dataPayload;
     final int lastUpdated;

     const CacheModel({
       required this.id,
       required this.dataPayload,
       required this.lastUpdated,
     });

     Map<String, Object?> toMap() {
       return {
         'id': id,
         'dataPayload': dataPayload,
         'lastUpdated': lastUpdated,
       };
     }

     factory CacheModel.fromMap(Map<String, Object?> map) {
       return CacheModel(
         id: map['id'] as int,
         dataPayload: map['dataPayload'] as String,
         lastUpdated: map['lastUpdated'] as int,
       );
     }
   }
   ```

3. **Initialize the Database**
   Implement a database service class to handle the connection and table creation. Use `path` to ensure cross-platform compatibility.
   ```dart
   import 'package:path/path.dart';
   import 'package:sqflite/sqflite.dart';

   class DatabaseService {
     static Database? _database;

     Future<Database> get database async {
       if (_database != null) return _database!;
       _database = await _initDB('app_cache.db');
       return _database!;
     }

     Future<Database> _initDB(String filePath) async {
       final dbPath = await getDatabasesPath();
       final path = join(dbPath, filePath);

       return await openDatabase(
         path,
         version: 1,
         onCreate: _createDB,
       );
     }

     Future<void> _createDB(Database db, int version) async {
       await db.execute('''
         CREATE TABLE cache_data(
           id INTEGER PRIMARY KEY,
           dataPayload TEXT NOT NULL,
           lastUpdated INTEGER NOT NULL
         )
       ''');
     }
   }
   ```

4. **Implement CRUD Operations**
   Add strict, immutable methods for database interactions. Always use `whereArgs` to prevent SQL injection.
   ```dart
   class CacheRepository {
     final DatabaseService _dbService = DatabaseService();

     Future<void> insertData(CacheModel data) async {
       final db = await _dbService.database;
       await db.insert(
         'cache_data',
         data.toMap(),
         conflictAlgorithm: ConflictAlgorithm.replace,
       );
     }

     Future<CacheModel?> getData(int id) async {
       final db = await _dbService.database;
       final maps = await db.query(
         'cache_data',
         where: 'id = ?',
         whereArgs: [id],
       );

       if (maps.isNotEmpty) {
         return CacheModel.fromMap(maps.first);
       }
       return null;
     }

     Future<void> deleteData(int id) async {
       final db = await _dbService.database;
       await db.delete(
         'cache_data',
         where: 'id = ?',
         whereArgs: [id],
       );
     }
   }
   ```

5. **Implement the Caching Strategy (Hit/Miss Logic)**
   **STOP AND ASK THE USER:** "What is the specific remote API endpoint or fetch logic you want to wrap with this caching layer, and what is the acceptable cache TTL (Time To Live)?"
   
   Once provided, implement the cache hit/miss logic:
   ```dart
   Future<CacheModel> fetchWithCache(int id, Duration ttl) async {
     final repository = CacheRepository();
     
     // Step 1: Check Cache
     final cachedData = await repository.getData(id);
     final now = DateTime.now().millisecondsSinceEpoch;

     if (cachedData != null) {
       final cacheAge = now - cachedData.lastUpdated;
       if (cacheAge < ttl.inMilliseconds) {
         // Cache Hit
         return cachedData;
       }
     }

     // Step 2: Cache Miss or Expired - Fetch from Remote
     try {
       final remoteDataPayload = await _fetchFromRemoteAPI(id); // Implement remote fetch
       
       final newData = CacheModel(
         id: id,
         dataPayload: remoteDataPayload,
         lastUpdated: now,
       );

       // Step 3: Save to Cache
       await repository.insertData(newData);
       return newData;
     } catch (e) {
       // Validate-and-Fix: Fallback to stale cache if network fails
       if (cachedData != null) {
         return cachedData;
       }
       rethrow;
     }
   }
   ```

6. **Validate and Fix**
   Instruct the agent to verify database locks or missing table exceptions. If a `DatabaseException` occurs indicating a missing table, trigger a schema migration or database reset routine.

## Constraints
*   **Never** use string interpolation for SQL queries (e.g., `where: "id = ${model.id}"`). **Always** use `whereArgs` to prevent SQL injection.
*   **Never** perform database I/O operations synchronously. Always use `async`/`await` to prevent blocking the main UI thread.
*   **Always** handle network failures gracefully during a cache miss by falling back to stale cached data if available.
*   **Do not** store large binary files (like images or videos) directly in SQLite; store the file path in SQLite and the binary data in the device's file system.
