---
layout: single
title: Setting up Services with Giraffe
date: 2017-08-11
categories: [FSharp]
tags: [Giraffe]
comments: true
share: true
---

If you ever look to see how to add user claims to your ASP.NET Core project, you might see guides saying to setup your services similar the following in your Startup.cs file

```cs
services.AddAuthorization(options =>
{
    options.AddPolicy("Administrators", policy => policy.RequireClaim(ClaimTypes.Role, "Administrator"));
    options.AddPolicy("Moderators", policy => policy.RequireClaim(ClaimTypes.Role, "Administrator", "Moderator"));
});
```

It is an interesting syntax but more importantly, how do you convert this to F# code?

In F#, it looks like this

```fsharp
services.AddAuthorization(fun options ->
    options.AddPolicy(
        "Administrators",
        ( fun policy -> policy.RequireClaim(ClaimTypes.Role, "Administrator") |> ignore )
    ) |> ignore
    options.AddPolicy(
        "Moderators",
        ( fun policy -> policy.RequireClaim(ClaimTypes.Role, "Administrator", "Moderator") |> ignore )
    ) |> ignore
) |> ignore
```

Inside a Giraffe project, the code above goes inside your ``configureServices`` function.