# Master Server Toolkit - UI Framework

## Overview

The UI framework in Master Server Toolkit provides a flexible system for creating and managing user interfaces in multiplayer games. The framework is built around the concept of **Views**, which are independent UI components that can be animated, validated and extended in a modular way.

## Main Components

### [Views System](Views.md)
The views system includes base classes for managing UI screens, windows and panels.

### [UI Components](Components.md)
Components for displaying data, properties, progress bars and other interface elements.

### [Validation System](Validation.md)
A validation system for user input with support for checking fields using regular expressions.

## Integration with MasterServerToolkit

The UI framework is tightly integrated with other MasterServerToolkit modules:

 - **Authentication UI**: Simple creation of registration and login forms
 - **Lobbies UI**: Interfaces for lobbies and rooms
 - **Matchmaking UI**: Components for finding games and creating matches
 - **Chat UI**: Interfaces for chat and messaging

## Getting Started

To use the UI framework:

1. Add the `ViewsManager` component to your scene
2. Create UI views by inheriting them from the `UIView` class
3. Register the views with unique IDs
4. Use `ViewsManager.Show()` and `ViewsManager.Hide()` to control the views

```csharp
// Get view by ID
var loginView = ViewsManager.GetView<UIView>("LoginView");

// Show the view
loginView.Show();

// Hide all views
ViewsManager.HideAllViews();
```

## Example Usage

```csharp
// Automatically show UI when a player logs in
authModule.OnUserLoggedInEvent += (user) => {
    // Hide the login screen
    ViewsManager.Hide("LoginView");
    // Show the main menu
    ViewsManager.Show("MainMenuView");
};
```
