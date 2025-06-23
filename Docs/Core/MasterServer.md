# Master Server Toolkit - Core

## Description
Master Server Toolkit (version 4.20.0) is a framework for creating dedicated servers and multiplayer games. The core consists of two main components: the `Mst` class and `MasterServerBehaviour`.

## Mst class
Central class of the framework providing access to all major systems.

### Main properties:
```csharp
// Framework version and name
Mst.Version     // "4.20.0"
Mst.Name        // "Master Server Toolkit"

// Main connection to the master server
Mst.Connection  // IClientSocket

// Advanced framework settings
Mst.Settings    // MstAdvancedSettings
```

### Main components:
```csharp
// Client methods
Mst.Client // MstClient - for game clients
Mst.Server // MstServer - for game servers

// Utilities and helpers
Mst.Helper // Helper methods
Mst.Security // Security and encryption
Mst.Create // Creating sockets and messages
Mst.Concurrency // Working with threads

// Systems
Mst.Events // Event channel
Mst.Runtime // Working with runtime data
Mst.Args // Command line arguments
```

## MasterServerBehaviour class

Singleton for managing the master server in Unity.

### Usage example:
```csharp
// Get instance
var masterServer = MasterServerBehaviour.Instance;

// Events
MasterServerBehaviour.OnMasterStartedEvent += (server) => {
    Debug.Log("Master Server started!");
};

MasterServerBehaviour.OnMasterStoppedEvent += (server) => {
    Debug.Log("Master Server stopped!");
};
```

### Key features:
- Automatic launch when the scene starts
- Singleton pattern for a single instance
- Command line arguments for IP and port
- Events for tracking server state

### Command line arguments:
```csharp
// Master server IP address
Mst.Args.AsString(Mst.Args.Names.MasterIp, defaultIP);

// Master server port
Mst.Args.AsInt(Mst.Args.Names.MasterPort, defaultPort);
```

## Quick start

1. Add MasterServerBehaviour to a scene
2. Configure IP and port
3. Run the scene or provide command line arguments

```bash
# Example launch with arguments
./MyGameServer.exe -masterip 127.0.0.1 -masterport 5000
```

## Important notes
- Each master server uses a single MasterServerBehaviour instance
- All components are initialized automatically when the Mst class is first used
- OnMasterStarted/Stopped events must be subscribed before the server starts
