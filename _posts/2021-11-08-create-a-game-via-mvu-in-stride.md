---
layout: single
title: Creating a game functionally with Stride
excerpt: Create a game in MVU style using the Stride Engine
date: 2021-11-08
modified: 2022-08-26
categories: [FSharp]
tags: [Game Jam, Stride, Game Dev]
comments: true
share: true
---

### Background

Inspired by the blog post where [Svetol ECS](https://www.sebaslab.com/svelto-miniexample-7-stride-engine-demo/) was implemented into Stride Engine. I wanted to create a custom implementation of the base Game class so that the game update loop would be better suited to a functional approach. I first attempted to use [Garnet](https://github.com/bcarruthers/garnet) but I don't have enough experience with ECS system to create an implementation. I then pivot to a MVU system which I am familiar with. The plan was to use the elmish library but I could not get the game engine update loop and elmish loop to work together so I rolled my own. The original proof of concept can be found [here](https://github.com/vtquan/MVU-Stride-Demo/tree/main/InitialProofOfConcept) but my implementation have changed since then. The following is about the current implementation.

### New Project

Starting from the basic solution created by Stride, add a new F# class library called `MyGame.Core`, I renamed `Library.fs` to `Program.fs`, and included all the Stride nuget packages from the `MyGame` project. Then add `MyGame` project as a reference to MyGame.Core. Then add `MyGame.Core` as a project reference in `MyGame.Windows`. Make sure to save the solution and make sure go to the Stride editor and select File -> Reload Project. Otherwise, the Stride editor will remove your F# project from the solution whenever changes are made. Another issue is that the Stride editor will try to build with fsc.exe and fail. You can build the F# project first through Visual Studio then run the project via the Editor or run the project entirely through Visual Studio.

![Setup]({{ site.url }}/assets/images/mvu-stride/fsc-error.png)

<figure class="video_container">
  <video controls="true" allowfullscreen="true" style="width: 100%;">
    <source src="{{ site.url }}/assets/images/mvu-stride/NewProject.webm" type="video/webm">
  </video>
</figure>

### Events

For the MVU system to work, I need some way to create and receive messages while the game is running. For this, I use the built in Event system in Stride. So inside the `MyGame` C# project, I create a static Events class consisted of a Player EventKey and a Player EventReceiver.

```csharp
using Stride.Engine;
using Stride.Engine.Events;
using System;

namespace MyGame
{
    public static class Events
    {
        public static readonly EventKey<string> PlayerEventKey = new();
        public static readonly EventReceiver<string> PlayerEventListener = new(PlayerEventKey, EventReceiverOptions.Buffered);
    }
}
```

I want to send `"Up"`, `"Down"`, `"Left"`, `"Right"`, and `"Stop"` messages so the event will be of type `string`.

It is important to set the event receiver to buffered. This will allows me to get all events that was fired instead of just the latest one.

### Player Component

The current scene will have only a Sphere Entity so let's create a MVU component for moving the ball. Create a new `Player.fs` above `Program.fs`. For my model, I want to hold the velocity of the ball and ball Enitity.

```fsharp
//Player.fs
namespace MyGame.Core

open Stride.Core.Mathematics    //Use Stride Vector3 instead of the built in F# version for better compatibility
open Stride.Engine;
open System.Linq

module Player =  
    type Model =
        {
            Velocity : Vector3
            Sphere : Entity
        }
```

For the message, I want to move the ball up, down, left, and right as well as stopping when no key is pressed so those will be my messages

```fsharp
type Msg =
    | Up
    | Down
    | Left
    | Right
    | Stop
```
Since I will receive `"Up"`, `"Down"`, `"Left"`, `"Right"`, and `"Stop"` messages, I need to create a map function to convert them.

```fsharp
let map message : Msg list = 
    match message with
    | "Left" -> [Left]
    | "Right" -> [Right]
    | "Up" -> [Up]
    | "Down" -> [Down]
    | "Stop" -> [Stop]
    | _ -> []
```

I also need to create an `empty` and `init` function. The `empty` function will be used to initialize the model at runtime. But since I need a reference to the Sphere entity that I can't get at runtime, I have an `init` function to set the proper value once the data is loaded. Once the scene is loaded can I get the reference to my Sphere entity. The init function also return a list of Msg for when an component need to send a message once initialized.

```fsharp
let empty =
    { Velocity = Vector3.Zero; Sphere = new Entity () }
        
let init (scene : Scene) : Model * Msg list =
    let sphere = scene.Entities.FirstOrDefault(fun x -> x.Name = "Sphere")     
    { empty with Velocity = Vector3.Zero; Sphere = sphere }, []
```

I like to reuse the `empty` Record to create the `init` Record because it saves me from repeating my code to initialize the value for my labels. This is especially handy for long Record.

For the View, I want to make the ball move depending on the velocity. The delta time is also passed so that movement is framerate independent. Since I am modifying the objects directly instead of creating a new view, it might be inappropriate to call this MVU.

```fsharp
let view model (deltaTime : float32) =
    model.Sphere.Transform.Position <- model.Sphere.Transform.Position + model.Velocity * deltaTime
```

Lastly, for Update, I want to update the model's Velocity based on the message.

```fsharp
let update msg model (deltaTime : float32) =
    match msg with        
    | Left -> 
        { model with Velocity = model.Velocity - Vector3.UnitX }, []                        
    | Right -> 
        { model with Velocity = model.Velocity + Vector3.UnitX }, []            
    | Up -> 
        { model with Velocity = model.Velocity - Vector3.UnitZ }, []            
    | Down -> 
        { model with Velocity = model.Velocity + Vector3.UnitZ }, []            
    | Stop -> 
        { model with Velocity = Vector3.Zero }, []
```

Copying from the Elmish library, my update function return a new model along with a list of Messages. This allows me to follow up with multiple messages. Also, I don't need to send a follow up message most of the time so I can return an empty list without having to create a special `Empty` message.

### Game Component

Just like the player component, we need a game component. This is a crucial part of the implementation since it is responsible for reading all the messages that was created. It is also responsible for calling the `view` and `update` functions of other components. Create the `Game.fs` class right below `Player.fs` and add the following code:

```fsharp
//Game.fs
namespace MyGame.Core

open Stride.Engine
open Stride.Engine.Events
open Stride.Games
open System.Linq

module Game =  
    type Model =
        {
            PlayerModel : Player.Model
        }

    type Msg =
        | PlayerMsg of Player.Msg

    let empty =
        { PlayerModel = Player.empty }
        
    let init (scene : Scene) : Model * Msg list =
        let playerModel, playerMsgs = Player.init scene
        let gameMsgs =
            [
                yield! List.map (PlayerMsg) playerMsgs
            ]
            |> List.distinct
        { empty with PlayerModel = playerModel }, gameMsgs

    let private mapEvent (eventReceiver : EventReceiver<'a>) (eventMap : 'a -> 'b list) (listMap : 'b list -> Msg list) =   
        let eventList = (Seq.empty).ToList()
        let numEvent = eventReceiver.TryReceiveAll(eventList)
        let events = Seq.toList eventList
        let messages =
            [
                for e in events do
                    yield! listMap (eventMap e)
            ]
        messages

    let mapAllEvent () : Msg list =
        let messages =
            [
                yield! mapEvent MyGame.Events.PlayerEventListener Player.map (List.map PlayerMsg)
            ] |> List.distinct
        messages

    let view (state : Model) (gameTime : GameTime) =
        let deltaTime = float32 gameTime.Elapsed.TotalSeconds

        Player.view state.PlayerModel deltaTime

    let update (state : Model) (cmds : Msg list) (gameTime : GameTime) =
        let deltaTime = float32 gameTime.Elapsed.TotalSeconds

        let updateFold ((state, msgs) : Model * Msg list) cmd  = 
            match cmd with
            | PlayerMsg(m) ->
                let (model,msg) = Player.update m state.PlayerModel deltaTime
                { state with PlayerModel = model }, msgs @ msg

        let newState, nextMessages = List.fold updateFold (state, []) cmds
        newState , List.distinct nextMessages
```

Like the Player component, there is the `Model`, `Msg`, `empty`, `init`, `view` and `update`. You can see how the view and update function of this class will call the respective functions for the Player component. The `view` and `update` function will need to be updated as more components are created.

The `Model` for the Game component will contain the model of the other component. In this case, there is only the Player model

The `Msg` will be a discriminated union of the other component msg.

The `init` function will call the `Player.init` function and also convert any Player Msg into the appropriate Game Msg.

`update` required a fold function and the accumulator is the state and follow up messages that can change after every update function for each message.

The `mapEvent` and `mapAllEvent` functions work together to read all event messages. `mapEvent` is designed to be generic so only `mapAllEvent` would need to be change to add new events.

>Let's take a look at the `mapEvent` function. This will take an `eventReceiver`, a `eventMap` function, and a `listMap` function. 
>
>The `eventReceiver` is the `PlayerEventListener` created inside the `Events` class of the `MyGame` project. 
>
>The `eventMap` is the `map` function inside the `Player` component class. This is so we can convert the string messages from the `PlayerEvent` into a type checked `Msg` for the `Player` component. 
>
>Lastly, the `listMap` is a basic `List.map` function since we need a list of `Game Msg` but we only have a list of `Player Msg` so we need to convert them. All this is so that `mapEvent` can be generic and can handle multiple events.

Thanks to the generic `mapEvent` function, more events can be added as needed like so:

```fsharp
let mapAllEvent () : Msg list =
    let messages =
        [
            yield! mapEvent MyGame.Events.PlayerEventListener Player.map (List.map PlayerMsg)
            //yield! mapEvent MyGame.Events.MusicEventListener Music.map (List.map MusicMsg)
            //yield! mapEvent MyGame.Events.SceneEventListener Scene.map (List.map SceneMsg)
        ] |> List.distinct
    messages
```

### A MVU Game Class    

Now that all the components are created, I need to modify the game update loop. Copying Svetol's implementation, I create a new class derived from the base Stride Game class inside the `Program.fs` file. 

```fsharp
//Program.cs
namespace MyGame.Core

open Stride.Engine;

open Game

type MvuGame() =
    inherit Game()

    let mutable State, Messages = Game.empty,[]
        
    override this.BeginRun () = 
        let mainScene = this.Content.Load<Scene>("MainScene")
        let state, messages = init mainScene
        State <- state
        Messages <- messages    
        
    override this.Update gameTime =
        base.Update(gameTime);
            
        Messages <- Messages @ mapAllEvent ()

        let newState, newMessages = update State Messages gameTime
        State <- newState
        Messages <- newMessages

        view State gameTime |> ignore

    override this.Destroy () =
        base.Destroy()
```

This class have two new members, Messages and State. Because State and Messages are class members, I can't pass in a Scene to grab the actual value until the BeginRun function is executed.

`BeginRun` is modified to call `Game.init` to initialize the `State` and `Messages` properties.

`Update` now call `Game.mapAllEvent` to get all event messages. It will then call `Game.update` to process all messages. In this implementation, I have it so that any follow up messages will be handled in the next frame update. Another implementation will have the update continued until no follow up message is received instead. However, this is simpler and prevent infinite loop from messages following each other.

### Creating a C# script

In the Stride Engine, nothing will happen without a script. We need a script to capture keyboard input and fire the appropriate events. Go in the editor and create a new Script called `SphereController` and put in the following.

```csharp
//MyGameApp.cs
using System;
using System.Collections.Generic;
using System.Linq;
using Stride.Input;
using Stride.Engine;

namespace MyGame
{
    public class SphereController : SyncScript
    {
        public List<Keys> KeysLeft { get; } = new List<Keys>();

        public List<Keys> KeysRight { get; } = new List<Keys>();

        public List<Keys> KeysUp { get; } = new List<Keys>();

        public List<Keys> KeysDown { get; } = new List<Keys>();

        public override void Update()
        {
            var isKeyPress = false;

            if (KeysLeft.Any(key => Input.IsKeyDown(key)))
            {
                isKeyPress = true;
                Events.PlayerEventKey.Broadcast("Left");
            }
            if (KeysRight.Any(key => Input.IsKeyDown(key)))
            {
                isKeyPress = true;
                Events.PlayerEventKey.Broadcast("Right");
            }
            if (KeysUp.Any(key => Input.IsKeyDown(key)))
            {
                isKeyPress = true;
                Events.PlayerEventKey.Broadcast("Up");
            }
            if (KeysDown.Any(key => Input.IsKeyDown(key)))
            {
                isKeyPress = true;
                Events.PlayerEventKey.Broadcast("Down");
            }
            if (isKeyPress == false)
            {
                Events.PlayerEventKey.Broadcast("Stop");
            }
        }
    }
}
```

This script will check on every frame if a specific key is pressed. If pressed, it will send the corresponding message. 

Assign the script to the Sphere entity. Assign the keys in the editor and move the camera up to a higher position to view the area.

<figure class="video_container">
  <video controls="true" allowfullscreen="true" style="width: 100%;">
    <source src="{{ site.url }}/assets/images/mvu-stride/Setup.webm" type="video/webm">
  </video>
</figure>

Finally, go to your platform project(`MyGame.Windows` in this case) and replace the following in MyGameApp.cs

```csharp
using (var game = new Game())
{
    game.Run();
}
```

with

```csharp
using (var game = new MyGame.Core.MvuGame())
{
    game.Run();
}
```

Run it either from Visual Studio or The Stride Editor for the following result

<figure class="video_container">
  <video controls="true" allowfullscreen="true" style="width: 100%;">
    <source src="{{ site.url }}/assets/images/mvu-stride/result.webm" type="video/webm">
  </video>
</figure>


The final project can found [here](https://github.com/vtquan/MVU-Stride-Demo/tree/main/BasicMvu) and the original proof of concept [here](https://github.com/vtquan/MVU-Stride-Demo/tree/main/InitialProofOfConcept).

### Closing Thoughts

One benefit of this approach is that it doesn't interfere with using the engine as designed. You can still create C# script and use all the editor functionality. You have a choice on whether to implement a feature in a functional or object oriented style. I have not run any benchmark so I don't know how this architecture will scale for large games. Something like a Data Oriented Programming approach with the Model containing data for multiple Entities and looping through each one on update is probably the ideal component. A good improvement is to add some multithreading. Two places that stand out is mapping all the event messages and running the update & view functions for each components. If performance is a major concern, the steps to modify the game class shown here are applicable to setting up an ECS architecture. 

Lastly, it is not shown here but it is common for component to send message to another component either in the update or view function. There are many ways to accomplish this in an MVU architecture but the easiest way for me is to call the Events inside the C# project to send messages like so:

```fsharp
let update msg model (deltaTime : float32) =
    match msg with        
    | Win -> 
        MyGame.Events.ScoreEventKey.Broadcast("Increase");
        model, []
```