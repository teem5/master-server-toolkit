# Master Server Toolkit - Documentation

## Description
Master Server Toolkit is a framework for creating multiplayer games using a client-server architecture. It provides ready-made modules for authentication, profiles, rooms, lobbies, chat and many other aspects of a multiplayer game.

## Main Modules

### Core System
- [Authentication](Modules/Authentication.md) - Authentication and user management
- [Profiles](Modules/Profiles.md) - User profiles and data management
- [Rooms](Modules/Rooms.md) - Room and game session system

### Game Modules
- [Achievements](Modules/Achievements.md) - Achievements system
- [Censor](Modules/Censor.md) - Unwanted content filtering
- [Chat](Modules/Chat.md) - Chat and messaging system
- [Lobbies](Modules/Lobbies.md) - Pre-game lobby system
- [Matchmaker](Modules/Matchmaker.md) - Game matching and filtering
- [Notification](Modules/Notification.md) - Notification system
- [Ping](Modules/Ping.md) - Connection check and latency measurement
- [QuestsModule](Modules/QuestsModule.md) - Quests and tasks system
- [WorldRooms](Modules/WorldRooms.md) - Persistent world areas

### Infrastructure
- [Spawner](Modules/Spawner.md) - Game server launching
- [WebServer](Modules/WebServer.md) - Built-in web server for API and admin panel

### Analytics and Monitoring
- [AnalyticsModule](Modules/AnalyticsModule.md) - Game event collection and analysis

### Tools
- [Tools](Tools/README.md) - Collection of helper utilities
  - [UI Framework](Tools/UI/README.md) - User interface system
  - [Attributes](Tools/Attributes.md) - Unity inspector extensions
  - [Terminal](Tools/Terminal.md) - Debug terminal
  - [Tweener](Tools/Tweener.md) - Animation utilities
  - [Utilities](Tools/Utilities.md) - Helper functions

## Module Structure

Each module typically consists of the following components:
1. **Server module** (`*Module.cs`) - server-side logic of the module
2. **Server implementation** (`*ModuleServer.cs`) - server API implementation
3. **Client side** (`*ModuleClient.cs`) - client API for communicating with the server
4. **Packets** (`Packets/*.cs`) - data structures exchanged between client and server
5. **Interfaces and models** - definition of contracts and data objects

## Getting Started

1. **Setting up the master server:**
   ```csharp
   // Add required modules
   var authModule = gameObject.AddComponent<AuthModule>();
   var profilesModule = gameObject.AddComponent<ProfilesModule>();
   var roomsModule = gameObject.AddComponent<RoomsModule>();

   // Configure database connection
   gameObject.AddComponent<YourDatabaseFactory>();
   ```

2. **Client setup:**
   ```csharp
   // Connect to server
   Mst.Client.Connection.Connect("127.0.0.1", 5000, (successful, error) => {
       if (successful)
           Debug.Log("Connection established");
   });

   // Use modules
   Mst.Client.Auth.SignIn(username, password, (successful, error) => {
       if (successful)
           Debug.Log("Authentication successful");
   });
   ```

## Best Practices

1. **Modular architecture** - add only the modules you need
2. **Use interfaces** - create your own implementations to integrate with your system
3. **Security** - always check permissions on the server side
4. **Scalability** - use the spawner system to balance the load
5. **Planning** - design module interaction in advance

## Additional Information

For more detailed information refer to the documentation of the respective modules.
