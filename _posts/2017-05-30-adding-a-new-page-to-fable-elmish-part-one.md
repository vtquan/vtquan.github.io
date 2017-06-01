---
layout: single
title: Adding a new page to Fable Elmish, Part 1
excerpt: Starting with a template
date: 2017-05-30
categories: [FSharp]
tags: [Fable, Elmish]
comments: true
share: true
---

### Creating a new page

Start with the new project you created from the [previous post]({% post_url 2017-05-25-getting-started-with-fable-elmish %}). This example will reuse the existing page so that we can focus on just adding the page to the project. 
First, navigate to the src folder and create a new folder named NewInfo. Then go into the Info folder and copy View.fs and paste it inside the NewInfo folder. Your folder structure should look like this

![Proper folder structure]({{ site.url }}images/adding-a-new-page-to-fable-elmish-part-one/folder-structure.png)

For the page to compiles properly, you need to add the page to your project. Open up the .fsproj in the root folder of your project and add the following code inside the <ItemGroup></ItemGroup> tag.

```xml
<Compile Include="src/NewInfo/View.fs" />
```

This is important. The .fsproj file shows the compile order of every files in the project. This means that the compiler compiles the first file you see first and then second. You can see the full content of my .fsproj file below

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
    <!-- NewInfo -->
    <Compile Include="src/NewInfo/View.fs" />
    <!-- Counter -->
    <Compile Include="src/Counter/Types.fs" />
    <Compile Include="src/Counter/State.fs" />
    <Compile Include="src/Counter/View.fs" />
    <!-- Home -->
    <Compile Include="src/Home/Types.fs" />
    <Compile Include="src/Home/State.fs" />
    <Compile Include="src/Home/View.fs" />
    <!-- Navbar -->
    <Compile Include="src/Navbar/View.fs" />
    <!-- App -->
    <Compile Include="src/Types.fs" />
    <Compile Include="src/State.fs" />
    <Compile Include="src/App.fs" />
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="FSharp.NET.Sdk" Version="1.0.*" PrivateAssets="All" />
    <DotNetCliToolReference Include="dotnet-fable" Version="1.0.4" />
  </ItemGroup>
  <Import Project=".paket\Paket.Restore.targets" />
</Project>
```

This tell you that the compiler compiles Global.fs first and then Info/View.fs. In general, you will want to add your file after

```xml
<!-- Global to the app -->
<Compile Include="src/Global.fs" />
```

and before

```xml
<!-- App -->
<Compile Include="src/Types.fs" />
<Compile Include="src/State.fs" />
<Compile Include="src/App.fs" />
```

Also, if your new page require code from another file, you need to add the file first and then your page. 
You might noticed that I include <!-- NewInfo --> in the code above. This is a comment and it is optional for you to add it in your code.

Now open the NewInfo/View.fs file and at the first line where it said 

```fs
module Info.View
```

and replace it with

```fs
module NewInfo.View
```

This change the domain of your file from Info.View to NewInfo.View. Then modify the page so that you can tell the difference from the info page. Change the following code

```fs
let root =
  div
    [ ClassName "content" ]
    [ h1
        [ ]
        [ str "About page" ]
      p
        [ ]
        [ str "This template is a simple application build with Fable + Elmish + React." ] ]
```

to

```fs

let root =
  div
    [ ClassName "content" ]
    [ h1
        [ ]
        [ str "New page" ]
      p
        [ ]
        [ str "This template is a simple application build with Fable + Elmish + React." ] ]
```

The root function return the html that will be shown when the page is loaded. What you just did is changing the header to display "New Page" instead of "About Page"

### Setting up the Handlers

You will now setup a link to your new page and create the code that serves it. 
First open up src/Global.fs and edit

```fs
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

```fs
type Page =
  | Home
  | Counter
  | About
  | NewPage

let toHash page =
  match page with
  | About -> "#about"
  | Counter -> "#counter"
  | Home -> "#home"
  | NewPage -> "#newpage"
```

Be aware that I made two changes. I add a NewPage union case to Page. That mean that Page could be either Home, Counter, About, or NewPage. I call it NewPage to point out that you do not need to name it the same as your page folder which is called NewInfo. 
Then I add a new pattern rule in the toHash function for the new union case. The "#newpage" represent what get added to the url when you open the page. 
For example, clicking on the Home link will change the url to [http://localhost:8080/#home](http://localhost:8080/#home). In summary, you add a new type of Page called NewPage and that the url for it is "#newpage"

Now to add your link to the side menu. First open up src/App.fs and edit 

```fs
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

```fsharp
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
          menuItem "About" Page.About currentPage 
          menuItem "New Page" Page.NewPage currentPage ] ]
```

This will add a link to your page inside the side menu. Watch out for the closing ]] and make sure you didn't duplicate them or put them at the wrong spot. The function menuItem is part of the template and it creates a link with the text "New Page" and it link to Page of type NewPage. 
The currentPage is so that the link would looks different when you are already looking at that page. I included the menuItem function code below so you can how it work

```fsharp
let menuItem label page currentPage =
    li
      [ ]
      [ a
          [ classList [ "is-active", page = currentPage ]
            Href (toHash page) ]
          [ str label ] ]
``` 

Still inside App.fs, edit your root function from

```fsharp
let root model dispatch =

  let pageHtml =
    function
    | Page.About -> Info.View.root
    | Counter -> Counter.View.root model.counter (CounterMsg >> dispatch)
    | Home -> Home.View.root model.home (HomeMsg >> dispatch)
```

to

```fsharp
let root model dispatch =

  let pageHtml =
	function
	| Page.About -> Info.View.root
	| Counter -> Counter.View.root model.counter (CounterMsg >> dispatch)
	| Home -> Home.View.root model.home (HomeMsg >> dispatch)
	| NewPage -> NewInfo.View.root
```

Looking at the changes closely, I add a pattern rule for the NewPage union case I created. 
This code is checking the Page that it is getting and depending on the kind of Page, it calls a root function from different module. 
Now remember earlier, the top line of your NewInfo/View.fs was changed to

```fsharp
module NewInfo.View
```

This means that your module is NewInfo.View. Also inside the NewInfo/View.fs file is a root function.  
To copy the other code, you call your root function with the module name to get NewInfo.View.root

Finally open up src/State.fs and modify

```fsharp
let pageParser: Parser<Page->Page,Page> =
  oneOf [
    map About (s "about")
    map Counter (s "counter")
    map Home (s "home")
  ]
```

to

```fsharp
let pageParser: Parser<Page->Page,Page> =
  oneOf [
    map About (s "about")
    map Counter (s "counter")
    map Home (s "home")
    map NewPage (s "newpage")
  ]
```

Now open your console with the directory at the root of your project folder. Run the following command and navigate to [http://localhost:8080/](http://localhost:8080/) and click on the link on the side to see your new page.

```
dotnet fable npm-run start
```