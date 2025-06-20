# Master Server Toolkit - Networking

## Architecture Overview

The MST networking system is built on a flexible architecture with support for different transport protocols (WebSockets, TCP). It provides reliable and secure data transfer between clients and servers.

```
Networking
├── Core Interfaces (IPeer, IClientSocket, IServerSocket)
├── Messages (IIncomingMessage, IOutgoingMessage)
├── Serialization
├── Transport Layer (WebSocketSharp)
└── Extensions
```

## Main Components

### Client-Server Interaction

#### IPeer
Represents a connection between the server and a client.

```csharp
// Main properties
int Id { get; }                    // Unique peer ID
bool IsConnected { get; }          // Connection status
DateTime LastActivity { get; set; } // Time of last activity

// Events
event PeerActionHandler OnConnectionOpenEvent;   // Connection opened
event PeerActionHandler OnConnectionCloseEvent;  // Disconnection

// Extensibility
T AddExtension<T>(T extension);    // Add extension
T GetExtension<T>();               // Get extension
```

#### IClientSocket
Client socket for connecting to a server.

```csharp
// Main properties
ConnectionStatus Status { get; }   // Connection status
bool IsConnected { get; }          // Whether connected
string Address { get; }            // IP address
int Port { get; }                  // Port

// Connection methods
IClientSocket Connect(string ip, int port, float timeoutSeconds);
void Close(bool fireEvent = true);

// Message handling
IPacketHandler RegisterMessageHandler(ushort opCode, IncommingMessageHandler handler);
```

#### IServerSocket
Server socket for listening for connections.

```csharp
// Security
bool UseSecure { get; set; }                 // Use SSL
string CertificatePath { get; set; }         // Certificate path
string CertificatePassword { get; set; }     // Certificate password

// Events
event PeerActionHandler OnPeerConnectedEvent;     // Client connected
event PeerActionHandler OnPeerDisconnectedEvent;  // Client disconnected

// Listening methods
void Listen(int port);                          // Local listening
void Listen(string ip, int port);               // Listen on IP:Port
```

### Messaging System

#### IIncomingMessage
Represents an incoming message.

```csharp
// Main properties
ushort OpCode { get; }             // Operation code
byte[] Data { get; }               // Message data
IPeer Peer { get; }                // Sender

// Data reading methods
string AsString();                // Read data as string
int AsInt();                      // Read data as int
float AsFloat();                  // Read data as float
T AsPacket<T>();                  // Deserialize to packet
```

#### IOutgoingMessage
Represents an outgoing message.

```csharp
// Main properties
ushort OpCode { get; }             // Operation code
byte[] Data { get; }               // Message data

// Send methods
void Respond();                   // Send empty response
void Respond(byte[] data);        // Send binary data
void Respond(ISerializablePacket packet); // Send packet
void Respond(ResponseStatus status);      // Send status
```

#### MessageFactory
Factory for creating messages.

```csharp
// Message creation
IOutgoingMessage Create(ushort opCode);                 // Empty message
IOutgoingMessage Create(ushort opCode, byte[] data);    // With binary data
IOutgoingMessage Create(ushort opCode, string data);    // With string
IOutgoingMessage Create(ushort opCode, int data);       // With number
```

### Data Serialization

#### ISerializablePacket
Interface for serializable packets.

```csharp
// Required methods
void ToBinaryWriter(EndianBinaryWriter writer); // Serialize
void FromBinaryReader(EndianBinaryReader reader); // Deserialize
```

#### SerializablePacket
Base class for serializable packets.

```csharp
// Implements ISerializablePacket
// Provides basic methods for inheritors
```

#### Serializer
Utilities for binary serialization.

```csharp
byte[] Serialize(object data);    // Serialize object to bytes
T Deserialize<T>(byte[] bytes);   // Deserialize bytes to object
```

## Transport Layer

### WebSocketSharp

```csharp
// Client socket
WsClientSocket clientSocket = new WsClientSocket();
clientSocket.Connect("127.0.0.1", 5000);

// Server socket
WsServerSocket serverSocket = new WsServerSocket();
serverSocket.Listen(5000);
```

## Usage Patterns

### Registering Handlers

```csharp
// Register handler
client.RegisterMessageHandler(MstOpCodes.Ping, OnPingMessageHandler);

// Handler
private void OnPingMessageHandler(IIncomingMessage message)
{
    // Send response
    message.Respond("Pong");
}
```

### Sending Messages

```csharp
// Simple send
client.SendMessage(MstOpCodes.Ping);

// Send with data
client.SendMessage(MstOpCodes.Auth, "username:password");

// Send packet
client.SendMessage(MstOpCodes.JoinLobby, new JoinLobbyPacket{ LobbyId = 123 });

// Send and wait for response
client.SendMessage(MstOpCodes.GetRooms, (status, response) =>
{
    if (status == ResponseStatus.Success)
    {
        var rooms = response.AsPacketsList<RoomOptions>();
        ProcessRooms(rooms);
    }
});
```

### Creating Packets

```csharp
public class PlayerInfoPacket : SerializablePacket
{
    public int PlayerId { get; set; }
    public string Name { get; set; }
    public float Score { get; set; }

    public override void ToBinaryWriter(EndianBinaryWriter writer)
    {
        writer.Write(PlayerId);
        writer.Write(Name);
        writer.Write(Score);
    }

    public override void FromBinaryReader(EndianBinaryReader reader)
    {
        PlayerId = reader.ReadInt32();
        Name = reader.ReadString();
        Score = reader.ReadFloat();
    }
}
```

## Security and Performance

### Secure Connections (SSL/TLS)

```csharp
// Configure SSL client
clientSocket.UseSecure = true;

// Configure SSL server
serverSocket.UseSecure = true;
serverSocket.CertificatePath = "certificate.pfx";
serverSocket.CertificatePassword = "password";
```

### Timeout Management

```csharp
// Set connection timeout
client.Connect("127.0.0.1", 5000, 10); // 10 second timeout

// Set response timeout
client.SendMessage(opCode, data, callback, 5); // 5 second timeout
```

### Delivery Methods

```csharp
// Reliable delivery (guaranteed)
peer.SendMessage(message, DeliveryMethod.Reliable);

// Unreliable delivery (faster but no guarantee)
peer.SendMessage(message, DeliveryMethod.Unreliable);
```
