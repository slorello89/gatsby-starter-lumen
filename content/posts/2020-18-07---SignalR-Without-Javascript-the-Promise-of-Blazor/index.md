---
title: SignalR without Javascript, the Promise of Blazor
draft: false
template: post
slug: "/signalr-without-javascript-using-blazor-webassembly"
date: "2020-07-18T12:29:00Z"
category: "blazor"
socialImage: ./media/SignalRLogo.png
tags: 
    - blazor
    - signalR
description: "SignalR enables our clients to listen for server events in real-time. With the dawn of Blazor WebAssembly, the processing of these events has never been simpler. Check out this tutorial to see how to manage."
---

![headline](./media/SignalRLogo.png)

SignalR is the canonical client-side async notification library for ASP.NET. With it, we can build clients that are ultra-responsive to changing conditions on our servers. But SignalR has always had one major flaw. To use it, you needed to use JavaScript. That's fair, right? We're writing web clients, which are always running in the browser, so of course, we need to use JavaScript. We'll suffer without type safety because of the functionality we're getting. It's always been a necessary evil, and we've dealt with it as such.

With the dawn of Blazor, this age of compromise is over. We can manage all of the data transfers between our servers and clients straight out of CLR types! That is what we're going to be demonstrating now.

## Objectives

Let's concretize our objectives here. We're going to build a trivial Single-Paged-App(SPA) in Blazor wasm. That app will have a simple form to read inputs from users, and a table to see incoming messages.

## Prerequisites

* Latest .NET Core 3.1 SDK
* Visual Studio Code or the latest version of Visual Studio. I will be using VS Code for this

## Create Project

Navigate to your source directory in the console and run the following command.

`dotnet new blazorwasm -ho -n SignalRClr`

This will create a new directory `SignalRClr`, in that directory it will create a solution called `SignalRClr` and then projects:

* `SignalRClr.Shared.csproj`
* `SignalRClr.Server.csproj`
* `SignalRClr.Client.csproj`

Those projects are what they say they are. The `Shared` will be the shared models between the client and the server. The `client` is going to be the compiled wasm that ends up in our client's browser. The `Server` is our server-side code. Run `cd SignalRClr` to navigate into the project's folder, then run `code .` to open it in VS Code.

## Add Dependencies

We need to add a `Microsoft.AspNetCore.SignalR.Client` dependency to our `SignalRClr.Client` project, and a `Microsoft.AspNetCore.SignalR.Core` dependency to our `SignalRClr.Server` project. We will also need the `System.ComponentModel.Annotations` class for our Shared project. Cd into `Client` and run the following:

```text
dotnet add package Microsoft.AspNetCore.SignalR.Client
```

Then cd into the `Server` directory parallel to the `Client` directory and run the following:

```text
dotnet add package Microsoft.AspNetCore.SignalR.Core
```

Then cd into the `Shared` directory and run the following:

```text
dotnet add package System.ComponentModel.Annotations
```

## Build the Model

Add a file to our `SignalRClr.Shared` project folder called `Message.cs`. Here we'll add a simple message class that takes a UserName and Text. We'll annotate them to make both UserName and Text required.

```csharp
using System.ComponentModel.DataAnnotations;
namespace SignalRClr.Shared
{
    public class Message
    {
        [Required]
        public string UserName { get; set; }
        [Required]
        public string Text { get; set; }
    }
}
```

## Add Message Hub

In the `Server` Project, add a new folder called `Hubs`. In that folder, add a `MessageHub` class, we will add a new Hub class `MessageHub` that will only have one method `SendMessage` which will push the inbound message down to all of the HubConnection Client's using the `SendAsync` method.

```csharp
using Microsoft.AspNetCore.SignalR;
using SignalRClr.Shared;
using System.Threading.Tasks;
namespace SignalRClr.Server.Hubs
{
    public class MessageHub : Hub
    {
        public async Task SendMessage(Message message)
        {
            await Clients.All.SendAsync("ReceiveMessage", message);
        }
    }
}
```

## Configure Middleware

In the `Startup.cs` file, we'll need to add a couple of lines of code to enable the MessageHub. In `ConfigureServices` the line:

```csharp
services.AddSignalR();
```

Then in the `app.UseEndpoints`'s delegate in the `Configure` method, map the hub. Your UseEndpoints should look like:

```csharp
app.UseEndpoints(endpoints =>
{
    endpoints.MapRazorPages();
    endpoints.MapControllers();
    endpoints.MapFallbackToFile("index.html");
    endpoints.MapHub<Hubs.MessageHub>("/messageHub"); //Add this line
});
```

## Build the Frontend

We're going use the `Client/Pages/Index.razor` file for our frontend. Mind you, I'm not an expert on frontend development, so this will look simple.

![Frontend](./media/frontend.png)

So as you can see here, we have two sections, a Messages Table and a form that we'll use to send the messages.

### Pull in Dependencies

We need to add a reference to `Microsoft.AspNetCore.SignalR.Client` and `SignalRClr.Shared`. We also need to inject a `NavigationManager`, and to clean up the HubConnection we're going to use we'll need to implement `IDisposable`. We will declare all this for our `Index` component with the following:

```csharp
@page "/"
@using Microsoft.AspNetCore.SignalR.Client
@using SignalRClr.Shared
@inject NavigationManager NavigationManager
@implements IDisposable
```

### Add the Messages Table

We now need to add a Messages Table to our component. One of the cool things about razor/blazor is that we can do all of this with the CLR types that we want to use. In particular, our Message Model. We'll create a table; then, in the Table's body, we'll loop through a Messages Array that we will declare later, and add the UserName and the Text as table data.

```Html
<h2>Messages</h2>
<table class="table-active">
    <thead>
        <tr>
            <th>User Name</th>
            <th>Text</th>
        </tr>
    </thead>
    <tbody>
        @foreach(var message in _messages)
        {
            <tr>
                <td>@message.UserName</td>
                <td>@message.Text</td>
            </tr>
        }
    </tbody>
</table>
```

### Add Message Form

Next, we'll add an `EditForm` to provide it our `Message` model and add a `ValidSubmit` method to run when the form has successfully validated.

```html
<EditForm Model="@_message" OnValidSubmit="SendMessage">
    <DataAnnotationsValidator />
    <ValidationSummary />
    <h3>User Name</h3>
    <input @bind="@_message.UserName" placeholder="User Name" class="input-group-text" />
    <h3>Text</h3>
    <input @bind="@_message.Text" placeholder="User Name" class="input-group-text" />
    <br />
    <button class="btn btn-primary" type="submit">Send Message</button>
</EditForm>
```

### Add Component Code

We will need to add a `@code` block to our component now. This block is all the logic our page needs to execute. We will add a `_hubConnection` property, which will manage the sending and receiving messages from our server over SignalR. We will add a `_messages` list, which is simply the messages we've received from the server. And we'll add a `_message` field, which will simply be the model we use for sending messages. 

When initializing the Component, we will create the HubConnection, mapping it to the `/messageHub` URI we declared in our middleware. When that HubConnection gets a `Receive Message` signal, it will add the inbound message to our `_messages` collection and notify the component the state has changed. Naturally, when we get a valid submission from our form, we will submit that new message to the server. Finally, when finalizing the component, we will dispose of the `_hubConnection`. All the code to do this looks like:

```csharp
@code {
    private HubConnection _hubConnection;
    private List<Message> _messages = new List<Message>();
    private Message _message = new Message();

    protected override async Task OnInitializedAsync(){
        _hubConnection = new HubConnectionBuilder()
            .WithUrl(NavigationManager.ToAbsoluteUri("/messageHub"))
            .Build();

        _hubConnection.On<Message>("ReceiveMessage",
            (message)=>{
                _messages.Add(message);
                StateHasChanged();
        });
        await _hubConnection.StartAsync();
    }

    public async Task SendMessage()
    {
        await _hubConnection.SendAsync("SendMessage", _message);
        _message = new Message();
    }

    public void Dispose()
    {
        _ = _hubConnection?.DisposeAsync();

    }
}
```

## That's It

And that's it! Everything we need to do to manage simple messages in real time between clients. To run this project, you can run the `Server` project. Cd into the Server directory and run `dotnet run`, and it will launch the project. Navigate to `localhost:5000` (or wherever you configure it to run kestrel) in your browser, and you will see our little message program.

## Wrapping Up

Naturally, this was a simple example that explicitly shows how simple it is to use these frameworks together. But these are the building blocks of how Blazor can be used to wipe out the need for javascript when interacting with our models.

## Resources

* The source code from this demo can be found in [GitHub](https://github.com/slorello89/SignalRClr)