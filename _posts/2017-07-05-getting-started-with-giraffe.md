---
layout: single
title: Getting Started with Giraffe
date: 2017-07-06
categories: [FSharp]
tags: [Giraffe]
comments: true
share: true
---

[Giraffe](https://github.com/dustinmoris/Giraffe) is framework that allow you to take advantage of ASP.NET Core in a functional manner. 

### Prerequisite

You will need to make sure that the [.NET Core SDK][microsoft-sdk-core] is installed

### Downloading the template

The easier way to get started is by creating a basic project using a template. You can download the template by running the following in your console

```bash
dotnet new -i giraffe-template::*
```

If this is the first time, you run the dotnet commands, it will do basic setup first before downloading the template.

### Creating a project

After downloading the template, navigate to the directory you want your project to be in and run the following

```bash
dotnet new giraffe -n yourProjectName
```

Then download the required packages and libraries with

```bash
dotnet restore
```

### Running the project

Now run the following command to view your new site. By default, the site is at [http://localhost:5000/](http://localhost:5000/).

```bash
dotnet run
```