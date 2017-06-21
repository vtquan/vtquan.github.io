---
layout: single
title: Adding a new page to Fable Elmish, Part 3
excerpt: Adding a rock, paper, scissor game to your project
date: 2017-06-20
categories: [FSharp]
tags: [Fable, Elmish]
comments: true
share: true
---

Now that you know how to add both static and dynamic pages to your project, let's take a deeper look into how to add functionality to a page. For this example, the page will have a rock, paper, scisssor game. There will be 3 buttons to select your choice and the page will randomly select a choice and keep track of the results.

![The final result]({{ site.url }}/assets/images/adding-a-new-page-pt3/completed.png)

### Creating a new page

Run the following commands to create a new project called RockPaperScissorElmish

```
dotnet new fable-elmish-react -n RockPaperScissorElmish
cd RockPaperScissorElmish
yarn
dotnet restore
```

Create a new folder called "RPS" inside the the src folder to store our files. Since our page will have functionalities, create the following 3 files inside the RPS folder.

>State.fs
>
>Types.fs
>
>View.fs

To add the new files to our project. Modify "RockPaperScissorElmish.fsproj" to look like the following

```xml
<Project Sdk="FSharp.NET.Sdk;Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard1.6</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <!-- Global to the app -->
    <Compile Include="src/Global.fs" />
    <!-- Info -->
    <Compile Include="src/Info/View.fs" />
    <!-- Counter -->
    <Compile Include="src/Counter/Types.fs" />
    <Compile Include="src/Counter/State.fs" />
    <Compile Include="src/Counter/View.fs" />
    <!-- Home -->
    <Compile Include="src/Home/Types.fs" />
    <Compile Include="src/Home/State.fs" />
    <Compile Include="src/Home/View.fs" />
    <!-- Rock, Paper, Scissor -->
    <Compile Include="src/RPS/Types.fs" />
    <Compile Include="src/RPS/State.fs" />
    <Compile Include="src/RPS/View.fs" />
    <!-- Navbar -->
    <Compile Include="src/Navbar/View.fs" />
    <!-- App -->
    <Compile Include="src/Types.fs" />
    <Compile Include="src/State.fs" />
    <Compile Include="src/App.fs" />
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="FSharp.NET.Sdk" Version="1.0.*" PrivateAssets="All" />
    <DotNetCliToolReference Include="dotnet-fable" Version="1.0.8" />
  </ItemGroup>
  <Import Project=".paket\Paket.Restore.targets" />
</Project>
```

Now our project can compiles our files.

### Adding Page Functionalities

First open up "src/RPS/Types.fs" and edit the content to look like the following

``` 
module RPS.Types

type Model = { Win : int; Lost : int; Result: string }

type Msg =
  | Rock
  | Paper
  | Scissor
```

The first line is giving the name of the module for our file. Our new types, Model and Msg, are both inside the module ``RPS.Types``. The model is the data that the page have access to. For our page, we want to show the number of time the user win or lose. For that, we declare our Model to be a record with an int field for ``Win`` and ``Lost``. We also want to display a message saying the result so we have a string record field called ``Result``. ``Msg`` is the message that the user will sent. For the game, the user only need to send a Rock, Paper, or Scissor message.

To design our page, edit "src/RPS/View.fs" to 

```
module RPS.View

open Fable.Helpers.React
open Fable.Helpers.React.Props
open Types

let simpleButton txt action dispatch =
  div
    [ ClassName "column is-narrow" ]
    [ a
        [ ClassName "button"
          OnClick (fun _ -> action |> dispatch) ]
        [ str txt ] ]

let root model dispatch =
  div
    [ ClassName "content" ]
    [ h1
        [ ]
        [ str "Rock Paper Scissor Game" ]
      p
        [ ]
        [ str "Pick your selection!" ]
      br []
      simpleButton "Rock" Rock dispatch
      simpleButton "Paper" Paper dispatch
      simpleButton "Scissor" Scissor dispatch
      p
        []
        [ str (sprintf "Result: %s" model.Result)]
      p
        []
        [ str (sprintf "Game Stats: %i Wins, %i Lost" model.Win model.Lost) ] ]
```

First thing first, we need to name our module. For this file, the module is ``RPS.View``. 

The open statements let us access functions and types that we need. The two ``Fabel.Helpers`` statements let us create html from functions. The ``Types`` statement refer to the ``RPS.Types`` module we created earlier. We didn't said ``open RPS.Types`` because we declare the module to be ``RPS.View`` earlier so it knows to look for ``RPS.Types``.

The ``simpleButton`` function creates a button in html. The button's text is given by the ``txt`` argument. The ``OnClick (fun _ -> action |> dispatch)`` mean that when the user click the button, it sends a specific action (one of your message of either Rock, Paper, or Scissor) to an dispatch function. If you know ReactJS then this code should look very familiar to you. It produces html similar to the following

```html
<div class="column is-narrow">
  <a class="button" onclick='dispatch(action)'>txt</a>
</div>
```

The root function is will print the content of our page. It uses the earlier simpleButton function to create 3 buttons sending a Rock, Paper, or Scissor Message. After that, it uses the model to display the result and stats for wins and losses.

>Note that F# files is compiled from top to bottom. The simpleButton must be defined before the root function can use it. 

We can finally add functionality to our page. Edit "src/RPS/State.fs" to

``` 
module RPS.State

open Elmish
open Types

let init () : Model * Cmd<Msg> =
  { Win = 0; Lost = 0; Result = "" }, []

let update msg model : Model * Cmd<Msg> =
  let rnd = System.Random()

  let convertIntToChoice num =
    match num with
    | 1 -> Rock
    | 2 -> Paper
    | _ -> Scissor

  let cpuChoice = rnd.Next(1, 4) |> convertIntToChoice

  match msg, cpuChoice with
  | Rock, Scissor 
  | Paper, Rock
  | Scissor, Paper
    -> { model with Win = model.Win + 1; Result = "You win!" }, [] // You win
  | Rock, Paper
  | Paper, Scissor
  | Scissor, Rock
    -> { model with Lost = model.Lost + 1; Result = "You lose!" }, [] // You lose
  | _ 
    -> { model with Result = "Draw!" }, [] // You draw
```

Looking at the functions, the ``init`` function creates the initial model and list of messages for the page. Since the model is the amount of time the user win or lost the game, the value is 0 for both at the beginning. There is also no result since the game haven't started yet so ``Result`` is just an empty string. There's no need to send any message at this time so an empty list is created.

The ``update`` function it where all the processing is done. It creates a random number and converts it to a Rock, Paper, or Scissor message. Then it matchs the user message with the created message to find the result. It then clones the model with the appropriate fields updated. The new model is returned with blank command since we don't want do anything else until the user send another message.

### Adding the new Page, Model, and Message

As before, we add a new Page type to represent our page. Open "src/Global.fs" and modify the following

``` 
type Page =
  | Home
  | Counter
  | About

let toHash page =
  match page with
  | About -> "#about"
  | Counter -> "#counter"
  | Home -> "#home"
```

to this

``` 
module Global

type Page =
  | Home
  | Counter
  | About
  | RockPaperScissor

let toHash page =
  match page with
  | About -> "#about"
  | Counter -> "#counter"
  | Home -> "#home"
  | RockPaperScissor -> "#rpsgame"

```

Now to add our new message and model. Modify "src/Types.fs" from

``` 
type Msg =
  | CounterMsg of Counter.Types.Msg
  | HomeMsg of Home.Types.Msg

type Model = {
    currentPage: Page
    counter: Counter.Types.Model
    home: Home.Types.Model
  }
```

to

``` 
type Msg =
  | CounterMsg of Counter.Types.Msg
  | HomeMsg of Home.Types.Msg
  | GameMsg of RPS.Types.Msg

type Model = {
    currentPage: Page
    counter: Counter.Types.Model
    home: Home.Types.Model
    game: RPS.Types.Model
  }
```

I choose to name the message and model ``GameMsg`` and ``game`` respectively but it doesn't matter what the name is. Just be sure to use the same name when you reference them later.

### Adding a Link to your new Page

Let's add the link to the new page. Edit "src/App.fs" from

``` 
let menu currentPage =
  aside
    [ ClassName "menu" ]
    [ p
        [ ClassName "menu-label" ]
        [ str "General" ]
      ul
        [ ClassName "menu-list" ]
        [ menuItem "Home" Home currentPage
          menuItem "Counter sample" Counter currentPage
          menuItem "About" Page.About currentPage ] ]
```

so that it look like this

``` 
let menu currentPage =
  aside
    [ ClassName "menu" ]
    [ p
        [ ClassName "menu-label" ]
        [ str "General" ]
      ul
        [ ClassName "menu-list" ]
        [ menuItem "Home" Home currentPage
          menuItem "Counter sample" Counter currentPage
          menuItem "Rock Paper Scissor Game" RockPaperScissor currentPage
          menuItem "About" Page.About currentPage ] ]
```

### Creating Page Handlers Code

Let's get our link working. First, edit the root function in "src/App.fs" so that

``` 
let root model dispatch =

  let pageHtml =
    function
    | Page.About -> Info.View.root
    | Counter -> Counter.View.root model.counter (CounterMsg >> dispatch)
    | Home -> Home.View.root model.home (HomeMsg >> dispatch)
```

looks like

``` 
let root model dispatch =

  let pageHtml =
    function
    | Page.About -> Info.View.root
    | Counter -> Counter.View.root model.counter (CounterMsg >> dispatch)
    | Home -> Home.View.root model.home (HomeMsg >> dispatch)
    | RockPaperScissor -> RPS.View.root model.game (GameMsg >> dispatch)
```

Edit "src/State.fs" from

``` 
let pageParser: Parser<Page->Page,Page> =
  oneOf [
    map About (s "about")
    map Counter (s "counter")
    map Home (s "home")
  ]
```

to

``` 
let pageParser: Parser<Page->Page,Page> =
  oneOf [
    map About (s "about")
    map Counter (s "counter")
    map Home (s "home")
    map RockPaperScissor (s "rpsgame")
  ]
```

Now the application can load the right page. I want to remind you that "rpsgame" came from the toHash function in "src/Global.fs" 

### Maintaining the Application's Models and Messages

While still in "src/State.fs" edit

``` 
let init result =
  let (counter, counterCmd) = Counter.State.init()
  let (home, homeCmd) = Home.State.init()
  let (model, cmd) =
    urlUpdate result
      { currentPage = Home
        counter = counter
        home = home }
  model, Cmd.batch [ cmd
                     Cmd.map CounterMsg counterCmd
                     Cmd.map HomeMsg homeCmd ]
```

to

``` 
let init result =
  let (counter, counterCmd) = Counter.State.init()
  let (home, homeCmd) = Home.State.init()
  let (game, gameCmd) = RPS.State.init()
  let (model, cmd) =
    urlUpdate result
      { currentPage = Home
        counter = counter
        game = game        
        home = home }
  model, Cmd.batch [ cmd
                     Cmd.map CounterMsg counterCmd
                     Cmd.map GameMsg gameCmd
                     Cmd.map HomeMsg homeCmd ]
```

We are using the init function in the ``RPS.State`` module to get our intitial model and command for our page. The application will use them to create a record with all the different models and create a sequence of commands.

> Remember that command are just a container for messages.

Finally, while still in "src/State.fs", change the following

``` 
let update msg model =
  match msg with
  | CounterMsg msg ->
      let (counter, counterCmd) = Counter.State.update msg model.counter
      { model with counter = counter }, Cmd.map CounterMsg counterCmd
  | HomeMsg msg ->
      let (home, homeCmd) = Home.State.update msg model.home
      { model with home = home }, Cmd.map HomeMsg homeCmd
```

to

``` 
let update msg model =
  match msg with
  | CounterMsg msg ->
      let (counter, counterCmd) = Counter.State.update msg model.counter
      { model with counter = counter }, Cmd.map CounterMsg counterCmd
  | GameMsg msg ->
      let (game, gameCmd) = RPS.State.update msg model.game
      { model with game = game }, Cmd.map GameMsg gameCmd
  | HomeMsg msg ->
      let (home, homeCmd) = Home.State.update msg model.home
      { model with home = home }, Cmd.map HomeMsg homeCmd
```

This let the application updates the models and commands after every update function.

### Running your Program

Run your project now and see your new page.

The final code can be found [here](https://github.com/vtquan/Adding-Dynamic-Page-to-Fable-Elmish).