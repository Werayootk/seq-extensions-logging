# Seq.Extensions.Logging [![Build status](https://ci.appveyor.com/api/projects/status/h7r1hv3cpd6e2ou3?svg=true)](https://ci.appveyor.com/project/datalust/seq-extensions-logging) [![NuGet Pre Release](https://img.shields.io/nuget/vpre/Seq.Extensions.Logging.svg)](https://nuget.org/packages/Seq.Extensions.Logging) [![Join the chat at https://gitter.im/datalust/seq](https://img.shields.io/gitter/room/datalust/seq.svg)](https://gitter.im/datalust/seq)

[Seq](https://getseq.net) is a flexible self-hosted back-end for the ASP.NET Core logging subsystem (_Microsoft.Extensions.Logging_). Log events generated by the framework and application code are sent over HTTP to a Seq server, where the structured data associated with each event is used for powerful filtering, correlation, and analysis.

For an example, see [the _dotnetconf_ deep dive session](https://channel9.msdn.com/Events/dotnetConf/2016/ASPNET-Core--deep-dive-on-building-a-real-website-with-todays-bits).

![Screenshot](https://github.com/datalust/seq-extensions-logging/blob/master/asset/screenshot.png?raw=true)

This package makes it a one-liner to configure ASP.NET Core logging with Seq.

### Getting started

Add [the NuGet package](https://nuget.org/packages/seq.extensions.logging) to the `dependencies` section of your `project.json` file:

```json
    "dependencies": {
        "Seq.Extensions.Logging": "2.1.1"
    }
```

In your `Startup` class's `Configure()` method, call `AddSeq()` on the provided `loggerFactory`.

```csharp
    public void Configure(IApplicationBuilder app,
                        IHostingEnvironment env,
                        ILoggerFactory loggerFactory)
    {
        loggerFactory.AddSeq("http://localhost:5341");
```

The framework will inject `ILogger` instances into controllers and other classes:

```csharp
class HomeController : Controller
{
    readonly ILogger<HomeController> _log;
    
    public HomeController(ILogger<HomeController> log)
    {
        _log = log;
    }
    
    public IActionResult Index()
    {
        _log.LogInformation("Hello, world!");
    }
}
```

Log messages will be sent to Seq in batches and be visible in the Seq user interface. Observe that correlation identifiers added by the framework, like `RequestId`, are all exposed and fully-searchable in Seq.

### Logging with message templates

Seq supports the templated log messages used by _Microsoft.Extensions.Logging_. By writing events with _named format placeholders_, the data attached to the event preserves the individual property values.

```csharp
var fizz = 3, buzz = 5;
log.LogInformation("The current values are {Fizz} and {Buzz}", fizz, buzz);
```

This records an event like:

| Property | Value |
| -------- | ----- |
| `Message` | `"The current values are 3 and 5"` |
| `Fizz` | `3` |
| `Buzz` | `5` |

Seq makes these properties searchable without additional log parsing. For example, a filter expression like `Fizz < 4` would match the event above.

### Additional configuration

The `AddSeq()` method exposes some basic options for controlling the connection and log volume.

| Parameter | Description | Example value |
| --------- | ----------- | ------------- |
| `apiKey` | A Seq [API key](http://docs.getseq.net/docs/api-keys) to authenticate or tag messages from the logger | `"1234567890"` |
| `levelOverrides` | A dictionary mapping logger name prefixes to minimum logging levels | `new Dictionary<string,LogLevel>{ ["Microsoft"] = LogLevel.Warning }` |
| `minimumLevel` | The level below which events will be suppressed (the default is `Information`) | `LogLevel.Trace` |

### JSON configuration

The Seq server URL, API key and other settings can be read from JSON configuration if desired.

In `appsettings.json` add a `"Seq"` property:

```json
{
  "Seq": {
    "ServerUrl": "http://localhost:5341",
    "ApiKey": "1234567890",
    "MinimumLevel": "Trace",
    "LevelOverride": {
      "Microsoft": "Warning"
    }
  }
}
```

And then pass the configuration section to the `AddSeq()` method:

```csharp
        loggerFactory.AddSeq(Configuration.GetSection("Seq"));
```

### Dynamic log level control

The logging provider will dynamically adjust the default logging level up or down based on the level associated with an API key in Seq. For further information see the [Seq documentation](http://docs.getseq.net/docs/using-serilog#dynamic-level-control).

### Troubleshooting

> Nothing showed up, what can I do?

If events don't appear in Seq after pressing the refresh button in the _filter bar_, either your application was unable to contact the Seq server, or else the Seq server rejected the log events for some reason.

#### Server-side issues

The Seq server may reject incoming events if they're missing a required API key, if the payload is corrupted somehow, or if the log events are too large to accept.

Server-side issues are diagnosed using the Seq _Ingestion Log_, which shows the details of any problems detected on the server side. The ingestion log is linked from the _Settings_ > _Diagnostics_ page in the Seq user interface.

#### Client-side issues

If there's no information in the ingestion log, the application was probably unable to reach the server because of network configuration or connectivity issues. These are reported to the application through `SelfLog`.

Add the following line after the logger is configured to print any error information to the console:

```csharp
Seq.Extensions.Logging.SelfLog.Enable(Console.Error);
```

If the console is not available, you can pass a delegate into `SelfLog.Enable()` that will be called with each error message:

```csharp
Seq.Extensions.Logging.SelfLog.Enable(message => {
    // Do something with `message`
});
```

#### Troubleshooting checklist

 * Check the Seq _Ingestion Log_, as described in the _Server-side issues_ section above.
 * Turn on the `SelfLog` as described above to check for connectivity problems and other issues on the client side.
 * [Raise an issue](https://github.com/datalust/seq-extensions-logging/issues), ask for help on the [Seq support forum](http://docs.getseq.net/discuss) or email **support@getseq.net**.
 
### Migrating to Serilog

This package is based on a subset of the powerful [Serilog](https://serilog.net) library. Not all of the options supported by the Serilog and Seq client libraries are present in the _Seq.Extensions.Logging_ package. Migrating to the full Serilog API however is very easy:

 1. Install packages _Serilog_, _Serilog.Extensions.Logging_ and _Serilog.Sinks.Seq_.
 2. Follow the instructions [here](https://github.com/serilog/serilog-extensions-logging) to change `AddSeq()` into `AddSerilog()` with a `LoggerConfiguration` object passed in
 3. Add `WriteTo.Seq()` to the Serilog configuration as per [the example](https://github.com/serilog/serilog-sinks-seq) given for the Seq sink for Serilog
