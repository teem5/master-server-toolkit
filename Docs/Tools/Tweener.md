# Master Server Toolkit - Tweener

## Description
A flexible system for creating animated transitions, interpolations and delayed actions. Tweener smoothly changes values of different types (float, int, string) over time and allows you to build sequences of actions.

## Key Features

### Basic Tweener
Common class for managing and running all tween actions.

```csharp
// Start a simple action
TweenerActionInfo actionInfo = Tweener.Start(() => {
    // Return true when the action is complete
    return true;
});

// Cancel the action
actionInfo.Cancel();
// or
Tweener.Cancel(actionInfo);
// or by ID
Tweener.Cancel(actionId);

// Check if the action is running
bool isRunning = actionInfo.IsRunning;
// or
bool isRunning = Tweener.IsRunning(actionInfo);
// or by ID
bool isRunning = Tweener.IsRunning(actionId);

// Handle completion
actionInfo.OnComplete((id) => {
    Debug.Log($"Action {id} completed");
});
// or
Tweener.AddOnCompleteListener(actionInfo, (id) => {
    Debug.Log($"Action {id} completed");
});
```

### Tween.Float
Smooth interpolation of floating point values.

```csharp
// Transition from 0 to 1 over 2 seconds with linear interpolation
TweenerActionInfo actionInfo = Tween.Float(0f, 1f, 2f, Easing.Linear, (float value) => {
    // Update value during the animation
    myCanvasGroup.alpha = value;
});

// Delayed start
TweenerActionInfo actionInfo = Tween.Float(0f, 1f, 2f, Easing.Linear, (float value) => {
    myCanvasGroup.alpha = value;
}, 1f); // Delay of 1 second

// Transition using a custom animation curve
AnimationCurve curve = new AnimationCurve(
    new Keyframe(0, 0),
    new Keyframe(0.5f, 0.8f),
    new Keyframe(1, 1)
);

TweenerActionInfo actionInfo = Tween.Float(0f, 1f, 2f, curve, (float value) => {
    myCanvasGroup.alpha = value;
});

// Reverse animation (ping-pong)
TweenerActionInfo actionInfo = Tween.Float(0f, 1f, 2f, Easing.Linear, (float value) => {
    myCanvasGroup.alpha = value;
}, 0f, true);
```

### Tween.Int
Smooth interpolation of integer values.

```csharp
// Animate counter from 0 to 100 over 3 seconds
TweenerActionInfo actionInfo = Tween.Int(0, 100, 3f, Easing.OutQuad, (int value) => {
    scoreText.text = value.ToString();
});

// Countdown animation
TweenerActionInfo actionInfo = Tween.Int(10, 0, 5f, Easing.Linear, (int value) => {
    countdownText.text = value.ToString();
});
```

### Tween.String
Animated replacement of characters in a string.

```csharp
// Typewriter effect
TweenerActionInfo actionInfo = Tween.String("", "Hello, world!", 2f, (string value) => {
    dialogText.text = value;
});

// Masked input animation
TweenerActionInfo actionInfo = Tween.String("", "12345", 1f, (string value) => {
    passwordText.text = new string('*', value.Length);
});
```

### Helper Methods

#### Wait - wait for a specified time
```csharp
// Wait 2 seconds and then execute an action
TweenerActionInfo actionInfo = Tween.Wait(2f, () => {
    DoSomething();
});

// Waiting inside a sequence
StartCoroutine(WaitExample());

IEnumerator WaitExample()
{
    Debug.Log("Action 1");

    // Waiting with Tween
    var waitInfo = Tween.Wait(2f, null);
    yield return new WaitUntil(() => !waitInfo.IsRunning);

    Debug.Log("Action 2");
}
```

#### Sequence - chain of actions
```csharp
// Create a sequence of actions
Tween.Sequence()
    .Add(() => {
        Debug.Log("Step 1");
        return true;
    })
    .Wait(1f)
    .Add(() => {
        Debug.Log("Step 2");
        return true;
    })
    .Wait(1f)
    .Add(() => {
        Debug.Log("Step 3");
        return true;
    })
    .Start();
```

#### RepeatForever - endless loop
```csharp
// Pulse an object
Tween.RepeatForever(() => {
    return Tween.Sequence()
        .Add(Tween.Float(1f, 1.2f, 0.5f, Easing.InOutQuad, (value) => {
            transform.localScale = new Vector3(value, value, value);
        }))
        .Add(Tween.Float(1.2f, 1f, 0.5f, Easing.InOutQuad, (value) => {
            transform.localScale = new Vector3(value, value, value);
        }));
});
```

## Easing Functions

Tweener supports various easing functions to control the nature of transitions:

```csharp
// Linear interpolation
Easing.Linear

// Quadratic functions
Easing.InQuad
Easing.OutQuad
Easing.InOutQuad

// Cubic functions
Easing.InCubic
Easing.OutCubic
Easing.InOutCubic

// Bounce functions
Easing.InBounce
Easing.OutBounce
Easing.InOutBounce

// Elastic functions
Easing.InElastic
Easing.OutElastic
Easing.InOutElastic

// Interpolation curve from the editor
AnimationCurve customCurve = new AnimationCurve(...);
```

## Usage Examples

### Animating UI elements
```csharp
// Smoothly show a panel
void ShowPanel()
{
    panel.gameObject.SetActive(true);
    panel.transform.localScale = Vector3.zero;

    Tween.Sequence()
        .Add(Tween.Float(0f, 1f, 0.3f, Easing.OutBack, (value) => {
            panel.transform.localScale = new Vector3(value, value, value);
        }))
        .Add(Tween.Float(0f, 1f, 0.2f, Easing.OutQuad, (value) => {
            panel.GetComponent<CanvasGroup>().alpha = value;
        }))
        .Start();
}

// Smoothly hide the panel
void HidePanel()
{
    Tween.Sequence()
        .Add(Tween.Float(1f, 0f, 0.2f, Easing.InQuad, (value) => {
            panel.GetComponent<CanvasGroup>().alpha = value;
        }))
        .Add(Tween.Float(1f, 0f, 0.3f, Easing.InBack, (value) => {
            panel.transform.localScale = new Vector3(value, value, value);
        }))
        .Add(() => {
            panel.gameObject.SetActive(false);
            return true;
        })
        .Start();
}
```

### Camera animation
```csharp
// Smooth camera movement
void MoveCamera(Vector3 targetPosition, float duration)
{
    Vector3 startPosition = Camera.main.transform.position;

    Tween.Float(0f, 1f, duration, Easing.InOutQuad, (value) => {
        Camera.main.transform.position = Vector3.Lerp(startPosition, targetPosition, value);
    });
}

// Smooth change of camera FOV
void ZoomCamera(float targetFOV, float duration)
{
    float startFOV = Camera.main.fieldOfView;

    Tween.Float(0f, 1f, duration, Easing.OutQuad, (value) => {
        Camera.main.fieldOfView = Mathf.Lerp(startFOV, targetFOV, value);
    });
}
```

### Creating game effects
```csharp
// Damage flash effect
void DamageFlash(SpriteRenderer renderer)
{
    Color normalColor = renderer.color;
    Color flashColor = Color.red;

    Tween.Sequence()
        .Add(Tween.Float(0f, 1f, 0.1f, Easing.Linear, (value) => {
            renderer.color = Color.Lerp(normalColor, flashColor, value);
        }))
        .Add(Tween.Float(1f, 0f, 0.1f, Easing.Linear, (value) => {
            renderer.color = Color.Lerp(normalColor, flashColor, value);
        }))
        .Start();
}

// Pulse effect
void PulseEffect(Transform target)
{
    Vector3 originalScale = target.localScale;
    Vector3 targetScale = originalScale * 1.2f;

    Tween.Sequence()
        .Add(Tween.Float(0f, 1f, 0.3f, Easing.OutQuad, (value) => {
            target.localScale = Vector3.Lerp(originalScale, targetScale, value);
        }))
        .Add(Tween.Float(1f, 0f, 0.3f, Easing.InQuad, (value) => {
            target.localScale = Vector3.Lerp(originalScale, targetScale, value);
        }))
        .Start();
}
```

## Best Practices

1. **Cancel unfinished actions** when objects are destroyed or scenes change
2. **Group related animations** in sequences for better control
3. **Use appropriate easing functions** for different types of movement
4. **Keep references to TweenerActionInfo** so you can cancel or check status
5. **Handle completion** with OnComplete for sequential operations
6. **Avoid overusing endless loops** (RepeatForever) and remember to cancel them
7. **Use Wait** instead of WaitForSeconds in coroutines for consistency
