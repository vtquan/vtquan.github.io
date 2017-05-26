---
layout: single
title: Getting Started with Fable Elmish
excerpt: Starting with a template
date: 2017-05-25
modified: 2017-05-26
categories: [FSharp]
tags: [Fable, Elmish]
comments: true
share: true
---

### Getting the Tools

- [__.NET Core SDK__][microsoft-sdk-core]
- [__Node.js__][node]

### Creating a project from the template

To begin, you need to download the template. Run this command in your console to download the template. If you already have the template, running this command will update the template.

```
dotnet new -i "Fable.Template.Elmish.React::*"
```

Now change your console directory to the directory that you want to create your new project in. Then run

```
dotnet new fable-elmish-react -n yourprojectname
```

You should have created a new folder containing your project. With your project created, update the directory of your console to the directory of your new folder with

```
cd yourprojectname
```

Now you need to download your packages and libraries to run the project by running

```
yarn
dotnet restore
```

If you have an error with yarn not being a valid command, make sure that you install Node.js and restart your computer.
The commands take a long time to finish so be patient. It is finished if you are able to input new commands.

### Running the project

Now run the following command to run your new project. You can view your site by going to [http://localhost:8080/](http://localhost:8080/)

```
dotnet fable npm-run start
```

If you want to build the project instead, run this command

```
dotnet fable npm-run build
```

[microsoft-sdk-core]: https://www.microsoft.com/net/download/core
[node]: https://nodejs.org/en/