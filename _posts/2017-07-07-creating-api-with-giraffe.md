---
layout: single
title: Creating API with Giraffe
date: 2017-07-06
categories: [FSharp]
tags: [Giraffe]
comments: true
share: true
---

Nearly any platforms these days have HTTP services. This means that with one set of APIs, you can share it between all your different platforms like web sites and mobile apps. Let's take a look at how you can create some API with Giraffe.

### Creating the project

Let's make a new project called, GiraffeApi. 
If the prerequisites haven't been installed already, then look at the [previous post]({% post_url 2017-05-30-adding-a-new-page-pt1 %}). Otherwise run the following commands in the console

``` bash
mkdir GiraffeApi
cd GiraffeApi
dotnet new giraffe
dotnet restore
```

### Design

Since the basic Giraffe temple already have a Message model. I want to make a set of API that can get, insert, update, and delete a list of Messages. One change that I want to made is modifying the Message model to have an Id that I can search against. 

Change the code inside Models/Message.fs to

```
namespace GiraffeApi.Models

[<CLIMutable>]
type Message =
    {
        Id : int
        Text : string
    }
```

There are multiple ideas on how to set up a set of API. For this project, these are the list of API that I want to create

>GET http://localhost:5000/message/%i
>
>CREATE http://localhost:5000/message/
>
>UPDATE http://localhost:5000/message/%i
>
>DELETE http://localhost:5000/message/%i
>
*%i is a stand in for the id of the message of interest*

### Creating the Controller

I want to have a seperate file that contains all the functions for the API. In ASP.NET term, this file is called the controller. At the root of the project, create a folder named "Controllers" and inside the folder, create a file named "MessageController.fs" that will be the controller for our Message model.

Inside the file, add the following cod

```
module GiraffeApi.Controllers

open Microsoft.AspNetCore.Http
open Giraffe.HttpHandlers
open Giraffe.HttpContextExtensions
open GiraffeApi.Models
```

This set the module of the file and the prerequisite classes and libraries that we will need. This also add a reference to the Message model that was modified earlier.

Now, let's create the data for the api to uses

```
let mutable Messages = 
    [
        { Id = 1; Text = "First Message" };
        { Id = 2; Text = "Second Message" };
        { Id = 3; Text = "Third Message" };
        { Id = 4; Text = "Fourth Message" }
    ]
```

Normally, data is retrieved from the database so data would be created in the database instead of a list. To keep the project simple and focused on api themselves, I create a list of Messages for the api to retrieved and modify.

#### GET

The GET api have two functionalities. One is simply returning a list of all messages. The other is returning a specific message with a specific Id.

```
let getMessages ctx =
    negotiate Messages ctx

let findMessage id =
    let result = List.tryFind (fun message -> message.Id = id) Messages
    match result with
    | Some(message) -> negotiate message 
    | None -> setStatusCode 404 >=> text "Message not found"
```

The ``getMessages`` function will return a list of all messages. The ctx parameter comes from the function handling the routing which you will see later. The ``negotiate`` function will takes the Messages list that was created earlier and return it to the user as a JSON, XML, or plain text depending on the user preference. To be specific with the return type, use ``json``, ``xml``, or ``text``. For example, ``json Messages ctx``.

>One thing you need to watch out for is using a parameter-less function in your routing. For example, if I write my ``getMessages`` function like this
>```
>let getMessages () =
>    negotiate Messages
>```
>When getMessages is used in routing. It will get evaluated upon started and value will not changed. If I made any changed to the Messages list, navigating to the route will still return the same Messages list created on startup. The technical reason for this, as I understand it, is that the routing function is composing the different functions together and this process evaluate any paramter-less function to pass one to the next. If you ever have a code like ``let f1 = f2 () >> f3``, you can see this effect happening there too. This is useful if you want to do anything at runtime once.

The ``findMessage`` function is mainly basic list searching. The important part is ``setStatusCode 404 >=> text "Message not found"``. This set the status code to 404 and return a text response of "Message not found." It's important to have accurate status code so the user can clearly troubleshoot problem.

#### POST

For POST, we'll have only one function which will add a new message

```
let submitMessage () 
    fun (ctx : HttpContext) ->
        async {
            let! message = ctx.BindModel<Message>()

            Messages <- List.append Messages [{ Id = (List.maxBy (fun m -> m.Id) Messages).Id + 1; Text = message.Text }]

            return! negotiate Messages ctx
        }
```

``ctx.BindModel`` would read the body of the POST request and convert it to an object. ``BindModel`` can read JSON, XML, or form urlencoded payload, or query string. For only accepting a specific type, ``BindJson``, ``BindXml``, ``BindForm``, or ``BindQueryString`` can be used instead. ``ctx.BindModel`` return an async object so it is wrapped in an async statement. The ``fun (ctx : HttpContext) ->`` indicates a nested function. The route code, that you will see later, will pass a HttpContext to your function. You can rewritten this to not use nested function as followed

```
let submitMessage (ctx:HttpContext) =
    async {
        let! message = ctx.BindModel<Message>()

        Messages <- List.append Messages [{ Id = (List.maxBy (fun m -> m.Id) Messages).Id + 1; Text = message.Text }]

        return! negotiate Messages ctx
    }
```

* This version requires changing the route code from ``route "/" >=> submitMessage()`` to ``route "/" >=> submitMessage``

#### PUT

Let's add the ability to update a message

```
let putMessage id =
    fun (ctx : HttpContext) ->
        async {
            let! message = ctx.BindModel<Message>()

            let editMessage origMessage =
                match origMessage.Id = id with
                | true ->
                    { origMessage with Text = message.Text }
                | false ->
                    origMessage

            match List.exists (fun message -> message.Id = id) Messages with
            | true ->
                Messages <- List.map editMessage Messages
                printfn "%A" Messages
                return! negotiate Messages ctx
            | false ->
                return! (setStatusCode 404 >=> json "Message not found") ctx
        }
```

Nothing special is going on here. Similar with the create function with getting a message via ``ctx.BindModel`` and standard F# code to edit the Message list.


#### DELETE

Finally, to delete a message via Id

```
let deleteMessage id =
    printfn "%A" Messages
    match List.exists (fun message -> message.Id = id) Messages with
    | true ->
        Messages <- List.filter (fun message -> message.Id <> id) Messages
        setStatusCode 204
    | false ->
        setStatusCode 404 >=> text "Message not found"
```

>It is important to know that ``setStatusCode 204`` means "No Content" and trying to return any content, ie ``setStatusCode 204 >=> text "Can't delete"`` will cause an exception

### Configuring the route

Upon creating a new project, Giraffe defines its routes inside the webApp function in Program.fs. By default, it looks like this

```
let webApp = 
    choose [
        GET >=>
            choose [
                route "/" >=> razorHtmlView "Index" { Text = "Hello world, from Giraffe!" }
            ]
        setStatusCode 404 >=> text "Not Found" ]
```

. For this project I decide to go with the following.

```
let webApp = 
    choose [
        subRoute "/message" (
            choose [
                GET >=>
                    choose [
                        route "/" >=> getMessages       //GET http://localhost:5000/message/
                        routef "/%i" findMessage        //GET http://localhost:5000/message/%i
                    ]
                POST >=>
                    choose [
                        route "/" >=> submitMessage()   //POST http://localhost:5000/message/
                        routef "/%i" addMessage         //POST http://localhost:5000/message/%i
                    ]
                PUT >=>
                    choose [
                        routef "/%i" putMessage         // PUT http://localhost:5000/message/%i
                    ]
                DELETE >=>
                    choose [
                        routef "/%i" deleteMessage      // DELETE http://localhost:5000/message/%i
                    ]
            ]
        )
        setStatusCode 404 >=> text "Not Found" ]
```

I have the subroute set so that I don't have to specify "message" for the following routes. Some functions required an Id so I used ``routef`` to retrieves a value from the URL. Note that the syntax for route and routef is different. One way of writing the same code would be

```
let webApp = 
    choose [
        GET >=>
            choose [
                route "/messages/" >=> getMessages 
                routef "/messages/%i" findMessage 
            ]
        POST >=>
            choose [
                route "/messages/" >=> submitMessage()
            ]
        PUT >=>
            choose [
                routef "/messages/%i" putMessage 
            ]
        DELETE >=>
            choose [
                routef "/messages/%i" deleteMessage
            ]
        setStatusCode 404 >=> text "Not Found" ]
```

### Running the project

Now run ``dotnet run`` to run your project. You can use [Postman](https://www.getpostman.com/) or similar tools to test your APIs. 