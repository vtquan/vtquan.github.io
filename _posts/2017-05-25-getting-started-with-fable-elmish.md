---
layout: single
title: Getting Started with Fable Elmish
excerpt: Starting with a template
date: 2017-05-25
modified: 2017-05-25
categories: [F#]
tags: [Fable, Elmish]
comments: true
share: true
---

### Getting the Tools

[__.NET Core SDK__][microsoft-sdk-core]
[__Node.js__][node]

### Creating a project from the template

If you haven't download the template before or if you want to update the template. Run this command in your console

```
dotnet new -i "Fable.Template.Elmish.React::*"
```

Now while in your console currently in the path that you want to create your new project, run

```
dotnet new fable-elmish-react -n yourprojectname
```

With your project created, update the directory of your console to be the path of your new folder with

```
cd yourprojectname
```

Now you need to download your packages and libraries to run the project properly by running

```
yarn
dotnet restore
```
If you have an error with yarn not being a valid command, make sure that you install Node.js and restart your computer.
The commands can take a long time to finish so make sure that you are able to input new command before continuing on 

### Running the project

You can now run the following command to run your new project and you can view your site by going to [http://localhost:8080/](http://localhost:8080/)

```
dotnet fable npm-run start
```

If you want to build the project instead, run this command

```
dotnet fable npm-run build
```


[microsoft-sdk-core]: https://www.microsoft.com/net/download/core
[node]: https://nodejs.org/en/