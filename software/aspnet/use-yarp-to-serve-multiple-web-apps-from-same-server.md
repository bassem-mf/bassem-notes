# Use YARP to Serve Multiple Web Apps From the Same Server

<br />
<br />

## The Problem

ASP.NET Core applications typically use the in-process HTTP server "Kestrel". Different ASP.NET/Kestrel processes cannot listen on the same TCP socket (a socket is a combination of an IP address and a port number). So if the server machine has only one IP address, it can run only one ASP.NET application that is listening on the standard HTTP and HTTPS ports (80 and 443 respectively).

## Possible Solutions   

1. Assign more IP addresses to the server. One IP for each web application. This will increase the hosting cost especially if you need to host many small web apps.
2. Bind the web applications to non-standard ports. This is not a good solution if users are going to access these web applications by typing URLs in a browser.
3. Use a reverse proxy like YARP or Nginx. The reverse proxy can listen on the standard HTTP and HTTPS ports. And forward each request to the intended web application which can be determined by the "Host" header value, or path, or any other data in the request header. Each web application listens on a unique, non-standard port. And receives HTTP requests from the reverse proxy, not from the client directly. The web application returns the HTTP response to the proxy. And the proxy forwards the response to the client. This configuration is depicted in the diagram below.

![Web apps behind a reverse proxy on the same machine](/images/software/aspnet/web-apps-behind-reverse-proxy-on-same-machine.webp)

This post will explain how to implement the third solution using YARP as the reverse proxy.

## Create an Executable YARP Server

YARP is distributed as a .NET library (NuGet package). So we need to create an executable program that uses this library.

1. Create a new .NET project using the "web" template.

    ```bash
    dotnet new web
    ```

2. Add a reference to the "Yarp.ReverseProxy" NuGet package by running the following command.

    ```bash
    dotnet add package Yarp.ReverseProxy
    ```

3. Replace the contents of "Program.cs" with the following.

    ```csharp
    WebApplicationBuilder builder = WebApplication.CreateBuilder(args);
    IConfigurationSection reverseProxyConfig = builder.Configuration.GetSection("ReverseProxy");
    builder.Services.AddReverseProxy().LoadFromConfig(reverseProxyConfig);

    WebApplication app = builder.Build();
    app.MapReverseProxy();

    app.Run();
    ```

    * `AddReverseProxy()` registers several YARP services with the dependency injection container. These services are needed by the YARP middleware.
    * `LoadFromConfig()` loads the reverse proxy configuration from an `IConfiguration` instance. The configuration usually comes from a JSON file, but it can also come from any other configuration source.
    * `app.MapReverseProxy()` registers the routes that are going to be handled by the reverse proxy, and defines the processing pipeline (sequence of middleware) for these routes.

4. Build and publish the .NET application files to the local file system.

    ```bash
    dotnet publish -c Release
    ```

    Then upload the contents of "\<project root\>/bin/Release/\<.NET version\>/publish/" to your server.

## Configure Kestrel and YARP

Edit "appsettings.json" on the server to configure logging, host filtering, Kestrel endpoints, hostname to certificate mapping, reverse proxy routes and destinations. The following example configuration can be used as a guide.

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Yarp.ReverseProxy.Forwarder.HttpForwarder": "Warning"
    }
  },
  "AllowedHosts": "www.domain1.com;domain1.com;subdomain.domain2.com;*.domain3.com",
  "Kestrel": {
    "Endpoints": {
      "Http": {
        "Url": "http://*:80"
      },
      "Https": {
        "Url": "https://*:443",
        "Sni": {
          "www.domain1.com": {
            "Certificate": {
              "Path": "<path to fullchain file>",
              "KeyPath": "<path to private key file>"
            }
          },
          "domain1.com": {
            "Certificate": {
              "Path": "<path to fullchain file>",
              "KeyPath": "<path to private key file>"
            }
          },
          "subdomain.domain2.com": {
            "Certificate": {
              "Path": "<path to fullchain file>",
              "KeyPath": "<path to private key file>"
            }
          },
          "*.domain3.com": {
            "Certificate": {
              "Path": "<path to fullchain file>",
              "KeyPath": "<path to private key file>"
            }
          }
        }
      }
    }
  },
  "ReverseProxy": {
    "Routes": {
      "route-www.domain1.com": {
        "Match": {
          "Hosts": [ "www.domain1.com", "domain1.com" ]
        },
        "ClusterId": "cluster-App1",
        "Transforms": [
          { "RequestHeaderOriginalHost": "true" }
        ]
      },
      "route-subdomain.domain2.com": {
        "Match": {
          "Hosts": [ "subdomain.domain2.com" ]
        },
        "ClusterId": "cluster-App2",
        "Transforms": [
          { "RequestHeaderOriginalHost": "true" }
        ]
      },
      "route-any.domain3.com": {
        "Match": {
          "Hosts": [ "*.domain3.com" ]
        },
        "ClusterId": "cluster-App3",
        "Transforms": [
          { "RequestHeaderOriginalHost": "true" }
        ]
      }
    },
    "Clusters": {
      "cluster-App1": {
        "Destinations": {
          "destination-App1": {
            "Address": "http://localhost:5000"
          }
        }
      },
      "cluster-App2": {
        "Destinations": {
          "destination-App2": {
            "Address": "http://localhost:5001"
          }
        }
      },
      "cluster-App3": {
        "Destinations": {
          "destination-App3": {
            "Address": "http://localhost:5002"
          }
        }
      }
    }
  }
}
```

* The JSON property `AllowedHosts` enables the Host Filtering Middleware. The value is a semicolon-delimited list of host names which should be allowed by the middleware.
* The `Kestrel > Endpoints` section defines two endpoints: `Http` and `Https`. The `Url` property specifies the IP address and port number to bind to. `*` means all IPv4 and IPv6 addresses.
* The `Sni` section under the `Https` endpoint specifies the HTTPS options to use for each hostname. The example above only specifies the certificate to use. But it is possible to specify other options like the TLS version, HTTP version, and whether a client certificate is required.
* The `ReverseProxy` section configures YARP. It has two sub-sections: `Routes` and `Clusters`.
* Each route specifies the requests that it will match. In the example above, we match by hostnames. But it also possible to match by path, method, headers, or query parameters.
* Each route has a `ClusterId` property which refers to the name of an entry in the `Clusters` section.
* The route can have `Transforms` to modify (or prevent the modification of) parts of the request or response before forwarding. The example above uses `{ "RequestHeaderOriginalHost": "true" }` to specify that the incoming request Host header should be preserved while forwarding the request to the destination server.
* A cluster contains one or more destinations as well as the rules for selecting the destination. This can be used to implement load balancing or fail-over systems.

Kestrel and YARP can react to most of the configuration changes while the application is running without needing a restart.

## Run YARP on the Server Machine

On the server, use a service manager to run the .NET application on system startup as a daemon. The service manager should also restart the application if it crashes. On Linux, you will typically use "systemd".

Create a ".service" file under "/etc/systemd/system/" to specify how systemd should start and manage the .NET application process. The file can be named something like "yarp-server.service" and its content can be similar to the following example.

```ini
[Unit]
Description=YARP Reverse Proxy Server

[Service]
WorkingDirectory=/opt/YarpServer
ExecStart=/path/to/dotnet /opt/YarpServer/YarpServer.dll
Restart=always
# Restart service after 10 seconds if the dotnet service crashes
RestartSec=10
KillSignal=SIGINT
SyslogIdentifier=yarp-server
User=www-data
AmbientCapabilities=CAP_NET_BIND_SERVICE
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=DOTNET_NOLOGO=true

[Install]
WantedBy=multi-user.target
```

Make sure that the user ("www-data" in the example above) has read access to the certificate files needed by the reverse proxy. This may include private key files.

The line `AmbientCapabilities=CAP_NET_BIND_SERVICE` permits the unprivileged process (a process owned by a user other than "root") to bind to privileged ports (port numbers less than 1024). This is needed because the reverse proxy needs to bind to ports 80 and 443.

Use the `systemctl enable` command to enable the systemd service so the reverse proxy runs on system startup. And include the `--now` flag to also start the reverse proxy right away.

```bash
sudo systemctl enable --now yarp-server.service
```

Use the `systemctl status` command to check if the reverse proxy started successfully.

```bash
sudo systemctl status yarp-server.service
```

The output of the `systemctl status` command should say that the service is enabled and active (running). If the service is not active, this means that it failed to start.

To investigate why the reverse proxy could not be started, or to see its console output and the systemd logs at any time, use the `journalctl` command.

```bash
sudo journalctl -u yarp-server.service
```

## Update the Back-End Server/Application

The client request will be forwarded by the reverse proxy to the back-end server/application using plain HTTP. Regardless of the protocol that was used in the original client request. But the back-end will probably need to know the original request protocol to be able to generate URLs and to redirect insecure HTTP requests to HTTPS.

Also the back-end may need to get the client IP that the original request came from. Even though it does not have a direct TCP connection with the client.

The original request protocol and IP address are not lost when YARP forwards the request. YARP will put the original protocol in the `X-Forwarded-Proto` header value, and will put the original IP address in the `X-Forwarded-For` header value. The back-end needs to be updated to read this information from the headers.

If the back-end is an ASP.NET application, you can use the [`ForwardedHeadersMiddleware`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.httpoverrides.forwardedheadersmiddleware) to apply forwarded headers to their matching fields on the current request.

```csharp
app.UseForwardedHeaders(
    new ForwardedHeadersOptions
    {
        ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto
    }
);
```

[`app.UseForwardedHeaders()`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.builder.forwardedheadersextensions.useforwardedheaders) adds and configures the `ForwardedHeadersMiddleware` to set `HttpContext.Connection.RemoteIpAddress` using the `X-Forwarded-For` header value, and set `HttpContext.Request.Scheme` using the `X-Forwarded-Proto` header value. There is no need to set `HttpContext.Request.Host` using the `X-Forwarded-Host` header value because we configured YARP to copy the incoming request Host header to the forwarded request.

The `ForwardedHeadersMiddleware` should be either the first in the pipeline, or the second after the `ExceptionHandlerMiddleware`.

The YARP server we created will not redirect HTTP requests to HTTPS. And will not redirect non-www requests to www. If this functionality is needed, it should be implemented by the back-end.

## References

* [Getting Started with YARP](https://microsoft.github.io/reverse-proxy/articles/getting-started.html)
* [Host filtering with ASP.NET Core Kestrel web server](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel/host-filtering)
* [Configure endpoints for the ASP.NET Core Kestrel web server](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel/endpoints)
* [YARP Configuration Files](https://microsoft.github.io/reverse-proxy/articles/config-files.html)
* [YARP Request and Response Transforms](https://microsoft.github.io/reverse-proxy/articles/transforms.html)
* [Host ASP.NET Core on Linux with Nginx](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/linux-nginx)
* [systemd execution environment configuration (AmbientCapabilities)](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html#AmbientCapabilities=)
* [Linux capabilities](https://man7.org/linux/man-pages/man7/capabilities.7.html) "CAP_NET_BIND_SERVICE" section
* [Configure ASP.NET Core to work with proxy servers and load balancers](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/proxy-load-balancer)