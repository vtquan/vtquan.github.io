---
layout: single
title: Creating a game functionally with Stride
excerpt: Create a game in MVU style using the Stride Engine
date: 2021-11-08
categories: [FSharp]
tags: [Game Jam, Stride]
comments: true
share: true
---
### Background

While there are many 3D game engine in the .NET ecosystem, nearly all are developed with C# in mind. If one wants to uses F# in these engine, the easiest and most common set up is to create a F# class library within the solution where logic can be written and called from the C# project. The [Stride Engine](https://www.stride3d.net) is no difference in this regard but I was looking into it due to being relatively lightweight for its functionalities and how well it integrate with Visual Studio. However, I was not satified with having the basic F# class library. The gameflow is still dictated by the C# project and passing values back and forth between projects was creating more work for myself. That's when I saw the blog post where [Svetol ECS](https://www.sebaslab.com/svelto-miniexample-7-stride-engine-demo/) was implemented into Stride Engine. Even though it is still through a class library, by creating a custom implementation of the base Game class, I have more control over the game loop. I wanted to do the same thing with [Garnet](https://github.com/bcarruthers/garnet). However, with the terminology being very different between Garnet and Svetol, along with the fact that I have no experience with ECS system, I did not get very far in its implementation. I then pivot to a MVU system which I am familiar with. The plan was to use the elmish library but I could not get the game engine update loop and elmish loop to work together so I rolled my own. The original proof of concept can be found [here](https://github.com/vtquan/MVU-Stride-Demo/tree/main/InitialProofOfConcept) but my implementation have changed during the game jam. The following is about the current implementation.

### Model, View, Update

Starting from the basic solution created by stride. I created a new F# class library and include all the Stride nuget packages. Save the solution and make sure go to the Stride editor and select File -> Reload Project. Otherwise, the Stride editor will remove your F# project from the solution whenever changes are made. Another issue is that the Stride editor will try to build with fsc.exe which no longer exist in newer sdk. You can build the F# project first through Visual Studio then run the project via the Editor or run the project entirely through Visual Studio.

![Setup]({{ site.url }}/assets/images/mvu-stride/fsc-error.png)

<figure class="video_container">
  <video controls="true" allowfullscreen="true" style="width: 100%;">
    <source src="{{ site.url }}/assets/images/mvu-stride/NewProject.webm" type="video/webm">
  </video>
</figure>

This scene will have only a ball Entity so let's create a MVU system for moving the ball. For my model, I want to hold the velocity of the ball and ball Enitity.

```fsharp
open Stride.Core.Mathematics  //Use Stride Vector3 instead of the built in F# version for better compatibility

//...

type GameModel =
    {
        Velocity : Vector3
        Sphere : Entity
    }
```

For the View, I want to make the ball move depending on the velocity. The delta time is also passed so that movement is framerate independent. This is not the same MVU system as used in Fabulous and fable elmish where a new view is created entirely from the model and a powerful diff engine will find what have changed and only update those. Instead, I am directly updating what is necessary in the scene.

```fsharp
let view model (gameTime : GameTime) =
    model.Sphere.Transform.Position <- model.Sphere.Transform.Position + model.Velocity * (float32 gameTime.Elapsed.TotalSeconds)
```

For the message, I want to move the ball up, down, left, and right as well as stopping when no key is pressed so that will be my messages

```fsharp
type GameMsg =
    | Up
    | Down
    | Left
    | Right
    | Stop
```

Lastly, for Update, I want to update the model velocity based on the messages

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

Copying from the elmish library, my update function return a new model along with a list of Messages. The list of message let me follow up a given message with multiple messages. And since most of the time I don't want to pass a message as a follow up, I can return an empty list without having to create a special Empty message.

I also need to create an empty and init function. The empty function will be used to initialize a class member. Then I use the init function to set the proper value. 

```fsharp
open System.Linq

//...

let empty () =
    { Velocity = Vector3.Zero; Ball = new Entity () }, []

let init (scene : Scene) =
    let sphere = scene.Entities.FirstOrDefault(fun x -> x.Name = "Sphere")     
    { Velocity = Vector3.Zero; Sphere = sphere }, []
```

### A MVU Game Class    

Now that all the MVU functions are created, I need to make them fit inside the game update loop. Copying Svetol's implementation, I create a new class derived from the base Stride Game class. This class have three members, Messages, State, and NextMessages. Because State and Messages are class members, I can't pass in a Scene to grab the actual value until the BeginRun function is executed.

```fsharp
type MvuGame() =
    inherit Game()

    let mutable State, Messages = empty ()
    let mutable NextMessages : GameMsg list = []
    
    override this.BeginRun () = 
        let mainScene = this.Content.Load<Scene>("MainScene")
        let state, messages = init mainScene
        State <- state
        Messages <- messages
    
    override this.Update gameTime =
        ()

    override this.Destroy () =
        base.Destroy()
```

Before filling in the Update function, we need a helper function to calls the appropriate update function depending on the message passed in and captures all the followup messages.

```fsharp
member private this.GameUpdate (cmds : GameMsg list) (state : GameModel) (gameTime : GameTime) =
    let GameUpdateFold ((state, msgs) : GameModel * GameMsg list) cmd  = 
        let (model,messages) = update cmd state (float32 gameTime.Elapsed.TotalSeconds)
        model, msgs @ messages

    let newState, nextMessages = List.fold GameUpdateFold (State, []) cmds
    State <- newState
    NextMessages <- NextMessages @ (List.distinct nextMessages)
```

In this case, we have only one message type so we call the update directly. Normally, there will be a pattern match between the different message. An example would be like so

```fsharp
member private this.GameUpdate (cmds : GameMsg list) (state : GameModel) (gameTime : GameTime) =
    let GameUpdateFold ((state, msgs) : GameModel * GameMsg list) cmd  = 
        match cmd with
        | PlayerMsg(m) ->
            let (model,messages) = Player.update m state.PlayerModel (float32 gameTime.Elapsed.TotalSeconds)
            { state with PlayerModel = model}, msgs @ messages
        | PlatformMsg(m) ->
            let (model,messages) = Platform.update m state.PlatformModel (float32 gameTime.Elapsed.TotalSeconds)
            { state with PlatformModel = model}, msgs @ messages

    let newState, nextMessages = List.fold GameUpdateFold (State, []) cmds
    State <- newState
    NextMessages <- NextMessages @ (List.distinct nextMessages)
```

Next, we need to create a way to receive messages while the game is running. For this, I use the built in Event system in Stride. So inside the C# project, I create a static Events class consisted of an event and an event receiver. With my current implementation, it is easier to handles one event that is fired multiple times per frame rather than checking multiple events if it was fired. So my Event will be a generic event and I found that an event containing a string and the entity that fired the event is flexible enough for most situations.

```csharp
using Stride.Engine;
using Stride.Engine.Events;
using System;

namespace GameJam
{
    public static class Events
    {
        public static readonly EventKey<Tuple<string, Entity>> GameEventKey = new();
        public static readonly EventReceiver<Tuple<string, Entity>> gameListener = new(GameEventKey, EventReceiverOptions.Buffered);
    }
}
```

In this case, there is only one Entity that is already being track in my model so `EventKey<string>` would also suffice. I also set the event listener to buffered. This will allows me to get all events that was fired instead of just the latest one.

Back to the F# project. I need to process the event

```fsharp
open Stride.Engine.Events
    
let private TryReceiveAllEvent (eventReceiver : EventReceiver<'a>) =   
    let events = (Seq.empty).ToList()
        let numEvent = eventReceiver.TryReceiveAll(eventList)
        numEvent, events

let private ProcessGameEvent ((message, entity) : string * Entity) : GameMsg list = 
    match message with
    | "Left" -> [Left]
    | "Right" -> [Right]
    | "Up" -> [Up]
    | "Down" -> [Down]
    | "Stop" -> [Stop]
    | _ -> []

let ProcessAllGameEvent (eventReceiver : EventReceiver<string * Entity>) : GameMsg list =
    let numEvent, events = TryReceiveAllEvent eventReceiver
    match numEvent with
    | 0 -> []
    | _ ->
        let msgSeq =
            seq {
                for e in events do
                    yield! ProcessGameEvent e
            }
        msgSeq

let ProcessAllEvent () : GameMsg list=        
    let msgSeq =
        seq {
            yield! ProcessAllGameEvent MyGame.Events.gameListener  //Can be expanded with additional yield! process listener
        }

    List.ofSeq (Seq.distinct msgSeq)
```

If I have more experience with Generic, I am sure that ProcessAllGameEvent can be made generic and would make it easier for add additional Event in the games. This is also a good candidate for parallel processing.

With all that in place, the update function can now be implemented. In this implementation, I have it so that all followup messages will be handled in the next frame update. A more proper implementation will have the update continued until no followup messages is received rather handling the followup messages in the next frame. However, this implemtation is simpler and prevent infinite loop from messages following each other.

```fsharp
override this.Update gameTime =
    base.Update(gameTime)
    
    Messages <- Messages @ ProcessAllEvent ()

    this.GameUpdate Messages State gameTime
    view State gameTime |> ignore

    Messages <- NextMessages
    NextMessages <- []
```

The location of `base.Update(gametime)` is variable. Without going inspecting the engine pipeline, I am not sure if it is better to call it first before the view function or vice versa.

That is all for the F# portion of the project. The full file is listed below:

```fsharp
namespace MyGame.MvuGame

open Stride.Core.Mathematics
open Stride.Engine;
open Stride.Games;
open System
open System.Linq
open Stride.Engine.Events

module Game =
    type GameModel =
        {
            Velocity : Vector3
            Sphere : Entity
        }

    type GameMsg =
        | Up
        | Down
        | Left
        | Right
        | Stop
        
    let empty () =
        { Velocity = Vector3.Zero; Sphere = new Entity () }, []
        
    let init (scene : Scene) =
        let sphere = scene.Entities.FirstOrDefault(fun x -> x.Name = "Sphere")     
        { Velocity = Vector3.Zero; Sphere = sphere }, []
    
    let view model (gameTime : GameTime) =
        model.Sphere.Transform.Position <- model.Sphere.Transform.Position + model.Velocity * (float32 gameTime.Elapsed.TotalSeconds)
    
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

    let private ProcessGameEvent ((message, entity) : string * Entity) : GameMsg list = 
        match message with
        | "Left" -> [Left]
        | "Right" -> [Right]
        | "Up" -> [Up]
        | "Down" -> [Down]
        | "Stop" -> [Stop]
        | _ -> []
        
    let private TryReceiveAllEvent (eventReceiver : EventReceiver<'a>) =   
        let events = (Seq.empty).ToList()
        let numEvent = eventReceiver.TryReceiveAll(events)
        numEvent, events

    let ProcessAllGameEvent (eventReceiver : EventReceiver<string * Entity>) : GameMsg seq =
        let numEvent, events = TryReceiveAllEvent eventReceiver
        match numEvent with
        | 0 -> Seq.empty
        | _ ->
            let msgSeq =
                seq {
                    for e in events do
                        yield! ProcessGameEvent e
                }
            msgSeq
    
    let ProcessAllEvent () : GameMsg list=        
        let msgSeq =
            seq {
                yield! ProcessAllGameEvent MyGame.Events.gameListener  //Can be expanded with additional yield! process listener
            }

        List.ofSeq (Seq.distinct msgSeq)

    type MvuGame() =
        inherit Game()

        let mutable State, Messages = empty ()
        let mutable NextMessages : GameMsg list = []
        
        override this.BeginRun () = 
            let mainScene = this.Content.Load<Scene>("MainScene")
            let state, messages = init mainScene
            State <- state
            Messages <- messages

        member private this.GameUpdate (cmds : GameMsg list) (state : GameModel) (gameTime : GameTime) =
            let GameUpdateFold ((state, msgs) : GameModel * GameMsg list) cmd  = 
                let (model,messages) = update cmd state (float32 gameTime.Elapsed.TotalSeconds)
                model, msgs @ messages

            let newState, nextMessages = List.fold GameUpdateFold (State, []) cmds
            State <- newState
            NextMessages <- NextMessages @ (List.distinct nextMessages)

        
        override this.Update gameTime =
            base.Update(gameTime);
            
            Messages <- Messages @ ProcessAllEvent ()

            this.GameUpdate Messages State gameTime
            view State gameTime |> ignore

            Messages <- NextMessages
            NextMessages <- []

        override this.Destroy () =
            base.Destroy()
```

### Creating a C# script

In the Stride Engine, nothing will happen without a script. We need a script to capture keyboard input and fire the appropriate events. Go in the editor and create a new Script called `SphereController` and put in the following.

```csharp
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
                Events.GameEventKey.Broadcast(new Tuple<string, Entity>("Left", Entity));
            }
            if (KeysRight.Any(key => Input.IsKeyDown(key)))
            {
                isKeyPress = true;
                Events.GameEventKey.Broadcast(new Tuple<string, Entity>("Right", Entity));
            }
            if (KeysUp.Any(key => Input.IsKeyDown(key)))
            {
                isKeyPress = true;
                Events.GameEventKey.Broadcast(new Tuple<string, Entity>("Up", Entity));
            }
            if (KeysDown.Any(key => Input.IsKeyDown(key)))
            {
                isKeyPress = true;
                Events.GameEventKey.Broadcast(new Tuple<string, Entity>("Down", Entity));
            }
            if (isKeyPress == false)
            {
                Events.GameEventKey.Broadcast(new Tuple<string, Entity>("Stop", Entity));
            }
        }
    }
}
```

Then assign the script to the Sphere entity. Assign the keys in the editor and move the camera up to a higher position to view the area.

<figure class="video_container">
  <video controls="true" allowfullscreen="true" style="width: 100%;">
    <source src="{{ site.url }}/assets/images/mvu-stride/Setup.webm" type="video/webm">
  </video>
</figure>

Finally, go to your platform project(MyGame.Windows in this case) and replace the following in MyGameApp.cs

```csharp
using (var game = new Game())
{
    game.Run();
}
```

with

```csharp
using (var game = new MvuGame.Game.MvuGame())
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


The final project can found [here](https://github.com/vtquan/MVU-Stride-Demo/tree/main/BasicMvu) and the original proof of concept [here](https://github.com/vtquan/MVU-Stride-Demo/tree/main/InitialProofOfConcept) with smoother movement and a jump function.
