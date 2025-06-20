# Master Server Toolkit - Core System

## Overview
The core of Master Server Toolkit consists of basic components that provide fundamental functionality for building multiplayer games. These components interact closely, creating a flexible and extensible architecture.

## Core structure

### [MasterServer](MasterServer.md)
Central component that provides access to all systems through the `Mst` class. Contains main settings and server configuration.

### [Client](Client.md)
Client-side part of the system for connecting to the server, sending requests and handling responses. Includes a base class for client modules.

### [Server](Server.md)
Server-side part of the system for handling connections, registering message handlers and managing modules.

### [Database](Database.md)
Abstraction for working with different databases. Allows using various data stores without changing the core code.

### [Events](Events.md)
Event system for message exchange between components without tight dependencies. Forms the basis for a loosely coupled architecture.

### [Keys](Keys.md)
System of constants and keys for unified data access. Prevents typos when working with string identifiers.

### [Localization](Localization.md)
Localization system that supports multiple languages. Allows easy addition and modification of translations.

### [Logger](Logger.md)
Logging system for tracking application operation. Supports various log levels and formatting.

### [Mail](Mail.md)
System for sending email messages. Used for registration confirmation, password recovery and other purposes.

## Component interaction

![Архитектура ядра](../Images/core_architecture.png)

### Key relationships:

1. **MasterServer** содержит центральный класс `Mst`, который предоставляет доступ ко всем другим компонентам:
   ```csharp
   // Access to the client
   Mst.Client
   
   // Access to the server
   Mst.Server
   
   // Access to the event system
   Mst.Events
   ```

2. **Client and Server** use a common network messaging system but operate on different sides of the connection:
   ```csharp
   // On the client side
   Mst.Client.Connection.SendMessage(MstOpCodes.SignIn, credentials);
   
   // On the server side
   server.RegisterMessageHandler(MstOpCodes.SignIn, HandleSignIn);
   ```

3. **Database** is used by server modules to access data:
   ```csharp
   // Get database accessor
   var dbAccessor = Mst.Server.DbAccessors.GetAccessor<IAccountsDatabaseAccessor>();
   
   // Use accessor
   var account = await dbAccessor.GetAccountByUsername(username);
   ```

4. **Events** is used by all components for loosely coupled information exchange:
   ```csharp
   // Send event
   Mst.Events.Invoke("userLoggedIn", userId);
   
   // Subscribe to event
   Mst.Events.AddListener("userLoggedIn", OnUserLoggedIn);
   ```

5. **Logger** is used everywhere for debugging and monitoring:
   ```csharp
   // Logging events
   Mst.Logger.Debug("Connection established");
   Mst.Logger.Error($"Failed to connect: {error}");
   ```

6. **Keys** is used to standardize data access:
   ```csharp
   // Use keys
   properties.Set(MstDictKeys.USER_ID, userId);
   properties.Set(MstDictKeys.USER_NAME, username);
   ```

7. **Localization** provides multilingual support:
   ```csharp
   // Get localized string
   string welcomeText = Mst.Localization.GetString("welcome_message");
   ```

8. **Mail** is used by server modules to communicate with users:
   ```csharp
   // Send email
   await Mst.Server.Mail.SendEmailAsync(email, subject, body);
   ```

## Initialization process

1. Create an instance of `MasterServerBehaviour`
2. Define connection settings (IP, port, security)
3. Register required modules
4. Start the server or connect the client
5. Initialize all registered modules
6. Set up necessary message handlers

## Module design principles

The core provides base classes for creating modules:
- `BaseServerModule` - for server modules
- `BaseClientModule` - for client modules

Modules follow these principles:
1. **Single responsibility** - each module is responsible for one feature
2. **Loose coupling** - communicate via events and a common interface
3. **Extensibility** - ability to inherit and override base behaviour
4. **Dependence on abstractions** - use interfaces instead of concrete implementations

## Recommendations for working with the core

1. **Use events** instead of direct calls to reduce coupling
2. **Encapsulate logic** in specialized modules
3. **Follow the architecture** of the base components when creating your own
4. **Use base classes** instead of building from scratch
5. **Document interfaces** to simplify future maintenance
