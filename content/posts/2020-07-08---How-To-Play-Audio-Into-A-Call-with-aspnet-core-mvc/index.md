---
title: How to Play Audio into a Call with ASP.NET Core MVC
draft: false
template: post
slug: "/how-to-play-audio-into-a-call-with-asp-net-core-mvc"
date: "2020-08-07T16:29:00Z"
category: "Voice"
social image: /media/Dev_Play-Audio_1000x420.png
tags: 
    - PSTN
    - Calls
    - aspnetcore    
    - dotnet
description: "In this tutorial we walk through how to play audio into a call in ASP.NET Core mvc using the Vonage Voice API."
canonical: https://www.nexmo.com/blog/2020/08/07/how-to-play-audio-into-a-call-with-asp-net-core-mvc-dr
---

![title](/media/Dev_Play-Audio_1000x420.png)

When you are building applications that are voice-enabled, meaning they can make and receive phone calls, the most fundamental thing you need to be able to do is to play audio into the call programmatically. This serves as the basis for IVRs, an alert system that you're going to be connected to a call, as a prompt to do something, or even just an on-hold message. Without the ability to play audio into a call, there are few use cases for voice-enabled apps beyond voice-proxying. In this tutorial, we'll be exploring how to get off the ground playing audio into calls with [Vonage's Voice API](https://developer.nexmo.com/voice/voice-api/overview) and [ASP.NET Core MVC](https://docs.microsoft.com/en-us/aspnet/core/mvc/overview?view=aspnetcore-3.1).

## Jump Right to the Code

If you want to skip over this tutorial and just jump right to the code, it's all available in [GitHub](https://github.com/nexmo-community/play-audio-aspnet-mvc)

## Prerequisites

* We're going to need the latest .NET Core SDK, I'm using 3.1
* We're going to use Visual Studio Code for this tutorial. Of course, this will also work with Visual Studio and Visual Studio for Mac. There may just be some slightly different steps for setup and running.
* We're going to need to create a [Vonage API Account](https://dashboard.nexmo.com/sign-up). Make sure you save your `api_key` and `api_secret`
* We'll be testing this with [ngrok](https://ngrok.com/) - so go ahead and follow their instructions for setting it up.
* We're going to need [npm](https://www.npmjs.com/) to fetch the nexmo-cli

## Topic Overview

There are two methods that we are going to be talking through to play audio into a call.

1. When our application is called, it will return an [NCCO](https://developer.nexmo.com/voice/voice-api/ncco-reference) (Nexmo Call Control Object) telling Vonage what to play into the call.
2. We will be using the Vonage Voice API (VAPI) to place a call and play audio into the call that we create.

In both cases, we are going to be using audio streaming functionality. This allows us to just play an audio file into a call. However, I'd be remiss if I didn't point out that in addition to playing audio files into calls, there is no shortage of an ability to customize what's played into a request. Whether it's using the [Text-To-Speech(TTS)](https://developer.nexmo.com/voice/voice-api/guides/text-to-speech) API or using [websockets](https://developer.nexmo.com/voice/voice-api/guides/websockets) to play dynamic audio streams into a call.

## Setup the Nexmo CLI

With npm installed we can go ahead and install and configure the Nexmo CLI using:

```sh
npm install nexmo-cli -g
nexmo setup API_KEY API_SECRET
```

This will get the Nexmo CLI setup and ready to run.

## Run Ngrok

I'm going to be throwing everything on `localhost:5000`. Run ngrok to publicly access `localhost:5000`.

```sh
ngrok http --host-header=localhost:5000 5000
```

Take a note of the URL that ngrok is running on. In my case, it's running on `http://7ca005ad1287.ngrok.io`. This is going to be the base URL for my webhooks going forward.

## Create our Vonage Application

A Vonage Application is a construct that enables us to link route our numbers and webhooks easily. You can create an application in the [Vonage Dashboard](https://dashboard.nexmo.com/applications), or you can just make it now with the CLI.

```sh
nexmo app:create "AspNetTestApp" http://7ca005ad1287.ngrok.io/webhooks/answer http://7ca005ad1287.ngrok.io/webhooks/events
```

This is going to create a Vonage Application. It's going to then link all incoming calls to that application to the answer URL: `http://7ca005ad1287.ngrok.io/webhooks/answer`. All call events that happen on that application are going to be routed to `http://7ca005ad1287.ngrok.io/webhooks/events`. This command is going to print out two things.

1. Your application id - you can view this application id in the [Vonage Dashboard](https://dashboard.nexmo.com/applications)
2. Your application's private key. Make sure you take this and save this to a file - I'm calling mine `private.key`

### Link your Vonage Number to Your Application

When you create your account, you are assigned a Vonage number. You can see this in the [numbers section of the dashboard.](https://dashboard.nexmo.com/your-numbers) Or you could alternatively just run `nexmo number:list` in your console to list your numbers. Take you Vonage Number and your Application Id and run the following:

```sh
nexmo link:app VONAGE_NUMBER APPLICATION_ID
```

With this done, your calls are going to route nicely to your URL.

### Create Project

In your console, navigate to your source code directory and run the following command:

```sh
dotnet new mvc -n PlayAudioMvc
```

That will create a directory and project called `PlayAudioMvc`, run the cd command to change your directory to `PlayAudioMvc`, and run the following to install the Nexmo.Csharp.Client library.

```sh
dotnet add package nexmo.csharp.client
```

Run `code .` to open Visual Studio Code.

## Edit the Controller

### Add Using Statements

We're going to be piggy-backing off of the `HomeController.cs` file, open `Controllers\HomeController.cs` and add the following using statements to the top:

```csharp
using Microsoft.Extensions.Configuration;
using Nexmo.Api.Voice.Nccos.Endpoints;
using Nexmo.Api.Voice.Nccos;
using Nexmo.Api.Voice;
using Nexmo.Api.Request;
```

### Inject Configuration

We're going to be leveraging dependency injection to get some of the configurable items for our app, namely the appId and private key. To this end, add an `IConfiguration` field to the `HomeController`, then add an `IConfigurationParameter` to the constructor and assign that `IConfiguration` field to the parameter. Your constructor should now look like this. While we're up here, let's also add a constant to this class to point to an audio file on the web, there's a serviceable one that Vonage provides for test cases that we'll link to:

```csharp
const string STREAM_URL = "https://nexmo-community.github.io/ncco-examples/assets/voice_api_audio_streaming.mp3";

private readonly IConfiguration _config;

public HomeController(ILogger<HomeController> logger, IConfiguration config)
{
    _config = config;
    _logger = logger;
}
```

### Add Answer Endpoint

We're going to be addressing case 1: where we receive a call from a user, and we want to play an audio file into it. We're going to need to add an action to our controller that will return a JSON string. Add the following to our `HomeController` class:

```csharp
[HttpGet("/webhooks/answer")]
public string Answer()
{
    var streamAction = new StreamAction{
        StreamUrl = new string[]
        {
            STREAM_URL
        }
    };
    var ncco = new Ncco(streamAction);
    return ncco.ToString();
}
```

When someone dials in, Vonage is going to make a Get Request on this URL. This method leverages our Ncco builder to create an NCCO; we then convert the NCCO to a string and return it. This will return a JSON string that will look like this:

```json
[{"streamUrl":["https://nexmo-community.github.io/ncco-examples/assets/voice_api_audio_streaming.mp3"],"action":"stream"}]
```

### Add Dial Out

The next action we're going to need to add is an action to dial out. This is just a bit more complicated. It's going to need to get our appId and key out of the configuration. It also needs a number to call and a number to call from, your Vonage Number, then it will build a Voice Client, create a request structure and place the call:

```csharp
[HttpPost]
public IActionResult MakePhoneCall(string toNumber, string fromNumber)
{
    var appId = _config["APPLICATION_ID"];
    var privateKeyPath = _config["PRIVATE_KEY_PATH"];

    var streamAction = new StreamAction{ StreamUrl = new string[] { STREAM_URL }};
    var ncco = new Ncco(streamAction);

    var toEndpoint = new PhoneEndpoint{Number=toNumber};
    var fromEndpoint = new PhoneEndpoint{Number=fromNumber};

    var credentials = Credentials.FromAppIdAndPrivateKeyPath(appId, privateKeyPath);
    var client = new VoiceClient(credentials);
    var callRequest = new CallCommand { To = new []{toEndpoint}, From = fromEndpoint, Ncco= ncco};
    var call = client.CreateCall(callRequest);
    ViewBag.Uuid = call.Uuid;
    return View("Index");
}
```

## Add a Frontend

Going off the theme of just piggybacking off our Home Controller, we're also going to be piggybacking off our Home View. Open `Views\Home\Index.cshtml`, you can remove the boilerplate div that's in there. We're going to be adding a basic form that will post to our `MakePhoneCall` action, and when the action finishes, we will display the call UUID from our Phone call. With this in mind, let's add the following to our file:

```html
@using (Html.BeginForm("MakePhoneCall", "home", FormMethod.Post))
{
    <div class="form-vertical">
        <h4>Call<h4>
                @Html.ValidationSummary(true, "", new { @class = "text-danger" })
                <div class="form-group">
                    @Html.Label("To")
                    <div>
                        @Html.Editor("toNumber", new { htmlAttributes = new { @class = "form-control" } })
                    </div>
                </div>

                <div class="form-group">
                    @Html.Label("From")
                    <div>
                        @Html.Editor("fromNumber", new { htmlAttributes = new { @class = "form-control" } })
                    </div>
                </div>
                <div class="form-group">
                    <div class="col-md-offset-2 col-md-10">
                        <button type="submit">Send</button>
                    </div>
                </div>
    </div>
}
@if(@ViewBag.Uuid != null){
    <h2>Call UUID: @ViewBag.Uuid</h2>
}
```

## Configure your app

### Add Config Variables

Remember that we are using the `IConfiguration` to get our appId and our private key path. With that in mind, let's open up `appsettings.json` and add the following keys:

```json
"APPLICATION_ID":"APPLICATION_ID",
"PRIVATE_KEY_PATH":"C:\\path\\to\\your\\private.key"
```

### Configure Kestrel or IIS Express

As I'm using VS Code, my app is naturally going to use kestrel. Regardless of whether you are using kestrel or IIS Express, go into `properties\launchSettings.json` and from the `PlayAudioMvc`->`applicationUrl` drop the `https://localhost:5001` endpoint—since we are not using SSL with ngrok, and we're pointing to port 5000. If you are using IIS Express, in `iisSettings`->`iisExpress`, set the `applicationUrl` to `http://localhost:5000` and the `sslPort` to 0.

## Testing Your Application

With all this done, all you need to do is run the command `dotnet run` and your application will start up and be hosted on port 5000. All that's left to do now is to call your application—you can call it on your Vonage number and place a call from your application. You can place the call by navigating to localhost:5000 and filling out and submitting the form.

## Resources

* You can learn much more about the Voice API By checking out our [documentation website](https://developer.nexmo.com/voice/voice-api/overview)
* You can learn A LOT about working voice APIs,  particularly the NCCOS, by checking out our [NCCO reference](https://developer.nexmo.com/voice/voice-api/ncco-reference)
* All the code from this tutorial is available in [GitHub](https://github.com/nexmo-community/play-audio-aspnet-mvc)