# Serve Static Site With ASP.NET and Kestrel

1. Create a new .NET project using the "web" template.

    ```bash
    dotnet new web
    ```

2. Replace the contents of "Program.cs" with the following.

    ```csharp
    using Microsoft.AspNetCore.HttpOverrides;

    WebApplicationBuilder builder = WebApplication.CreateBuilder(args);
    WebApplication app = builder.Build();

    // The app.UseForwardedHeaders() call is needed only if this ASP.NET application will be
    // deployed as a back-end server behind a reverse proxy server
    app.UseForwardedHeaders(
        new ForwardedHeadersOptions
        {
            ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto
        }
    );

    if (!app.Environment.IsDevelopment())
        app.UseHsts();
    app.UseHttpsRedirection();

    app.UseFileServer();

    app.Run();
    ```

    [`app.UseForwardedHeaders()`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.builder.forwardedheadersextensions.useforwardedheaders) is needed only if this ASP.NET application will be deployed as a back-end server behind a reverse proxy server like YARP or Nginx. This call adds and configures the [`ForwardedHeadersMiddleware`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.httpoverrides.forwardedheadersmiddleware) to set `HttpContext.Connection.RemoteIpAddress` using the `X-Forwarded-For` header value, and set `HttpContext.Request.Scheme` using the `X-Forwarded-Proto` header value.

    [`app.UseFileServer()`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.builder.fileserverextensions.usefileserver) without any arguments adds two middleware:
    
    * [`StaticFileMiddleware`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.staticfiles.staticfilemiddleware) which enables the serving of static files placed under the "wwwroot" folder.
    * [`DefaultFilesMiddleware`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.staticfiles.defaultfilesmiddleware) rewrites URLs to enabled the serving of default files from "wwwroot" without requiring the request URL to include the file's name. So a request to "www.mywebsite.com" can return "wwwroot/index.html" if it exists. And a request to "www.mywebsite.com/dir" can return "wwwroot/dir/index.html" if it exists.

3. Trust the ASP.NET HTTPS development certificate if it is not already trusted.

    ```bash
    dotnet dev-certs https --trust
    ```

    If you are on Linux, this command may or may not work for your Linux distribution. If it does not work, you will need to figure out how to trust the development certificate manually.

4. Open "Properties/launchSettings.json" and delete the "http" profile.

    ```diff
    {
        "$schema": "https://json.schemastore.org/launchsettings.json",
        "profiles": {
    -        "http": {
    -            "commandName": "Project",
    -            "dotnetRunMessages": true,
    -            "launchBrowser": true,
    -            "applicationUrl": "http://localhost:5196",
    -            "environmentVariables": {
    -                "ASPNETCORE_ENVIRONMENT": "Development"
    -            }
    -        },
            "https": {
                "commandName": "Project",
                "dotnetRunMessages": true,
                "launchBrowser": true,
                "applicationUrl": "https://localhost:7103;http://localhost:5196",
                "environmentVariables": {
                    "ASPNETCORE_ENVIRONMENT": "Development"
                }
            }
        }
    }
    ```

    After deleting the "http" profile, the second "https" profile will be used by default.

    The "launchSettings.json" file is only used on the local development machine. It is not deployed to the production environment.

5. Create a "wwwroot" folder under the project root folder. And put the static site files in this folder.

6. Run the application on the development machine to make sure it is working correctly.

    ```bash
    dotnet run
    ```

7. Create a file named "appsettings.Production.json" in the project root folder. If this ASP.NET application will be deployed as a public-facing edge server, then use a JSON configuration similar to the following.

    ```json
    {
        "AllowedHosts": "www.example.com",
        "Kestrel": {
            "Endpoints": {
                "Http": {
                    "Url": "http://*:80"
                },
                "Https": {
                    "Url": "https://*:443",
                    "Certificate": {
                        "Path": "<path to fullchain file>",
                        "KeyPath": "<path to private key file>"
                    }
                }
            }
        }
    }
    ```

    If this ASP.NET application will be deployed as a back-end server behind a reverse proxy server like YARP or Nginx, and the reverse proxy is on the same machine, then use a JSON configuration similar to the following.

    ```json
    {
        "AllowedHosts": "www.example.com",
        "Kestrel": {
            "Endpoints": {
                "Http": {
                    "Url": "http://localhost:5000"
                }
            }
        }
    }
    ```

    Both the edge server configuration and the back-end server configuration enable the Host Filtering Middleware by defining the `AllowedHosts` property. This works for the back-end server scenario because, usually, reverse proxy servers preserve the `Host` header value while forwarding requests to the back-end server.

    The edge server needs an HTTPS endpoint to communicate securely with clients. And it also needs an HTTP endpoint to receive insecure requests and respond with redirects to secure URLs. The back-end server only needs an HTTP endpoint because the reverse proxy server will receive the HTTPS request from the client and will forward it as an HTTP request to the back-end server. There is no need to encrypt the communication between the reverse proxy and the back-end server when they are on the same machine.

    The edge server configuration uses the wildcard `*` to bind to all IPv4 and IPv6 addresses. The back-end server configuration uses the hostname `localhost` to bind to both IPv4 and IPv6 loopback interfaces.

    The edge server listens on the standard HTTP and HTTPS ports (80 and 443 respectively). The back-end server listens on a custom port number (5000 in the example configuration above) because the requests will come from the reverse proxy server which can be configured to use any port.

8. Build and publish the .NET application files to the local file system.

    ```bash
    dotnet publish -c Release
    ```

    Then upload the contents of "\<project root\>/bin/Release/\<.NET version\>/publish/" to your server.

9. On the server, use a service manager to run the ASP.NET application on system startup as a daemon. The service manager should also restart the application if it crashes. On Linux, you will typically use "systemd".

    Create a ".service" file under "/etc/systemd/system/" to specify how systemd should start and manage the ASP.NET application process. The file can be named something like "kestrel-www.example.com.service" and its content can be similar to the following example.

    ```ini
    [Unit]
    Description=.NET Static Site Server for the Hostname www.example.com

    [Service]
    WorkingDirectory=/var/www/www.example.com
    ExecStart=/path/to/dotnet /var/www/www.example.com/StaticSiteServer.dll
    Restart=always
    # Restart service after 10 seconds if the dotnet service crashes
    RestartSec=10
    KillSignal=SIGINT
    SyslogIdentifier=www.example.com
    User=exampleuser
    # CAP_NET_BIND_SERVICE is needed only if the server is a public-facing edge server that
    # will bind to ports 80 and 443
    AmbientCapabilities=CAP_NET_BIND_SERVICE
    Environment=ASPNETCORE_ENVIRONMENT=Production
    Environment=DOTNET_NOLOGO=true

    [Install]
    WantedBy=multi-user.target
    ```

    If you configured the application to be a public-facing edge server:

    * Make sure that the user ("exampleuser" in the example above) has read access to the certificate files needed by Kestrel. This may include private key files.
    * Include the line `AmbientCapabilities=CAP_NET_BIND_SERVICE` to permits the unprivileged process (a process owned by a user other than "root") to bind to privileged ports (port numbers less than 1024). This is needed because the application needs to bind to the standard HTTP and HTTPS ports (80 and 443 respectively).

    Use the `systemctl enable` command to enable the systemd service so the web application runs on system startup. And include the `--now` flag to also start the application right away.

    ```bash
    sudo systemctl enable --now kestrel-www.example.com.service
    ```

    Use the `systemctl status` command to check if the application started successfully.

    ```bash
    sudo systemctl status kestrel-www.example.com.service
    ```

    The output of the `systemctl status` command should say that the service is enabled and active (running). If the service is not active, this means that it failed to start.

    To investigate why the application could not be started, or to see the application's console output and the systemd logs at any time, use the `journalctl` command.

    ```bash
    sudo journalctl -u kestrel-www.example.com.service
    ```

10. If you configured the ASP.NET application to run as a back-end server, then you will need to [setup and configure a reverse proxy server](use-yarp-to-serve-multiple-web-apps-from-same-server.md).

## References

* [Static files in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/static-files)
* [Enforce HTTPS in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/security/enforcing-ssl)
* [Configure endpoints for the ASP.NET Core Kestrel web server](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel/endpoints)
* [Host filtering with ASP.NET Core Kestrel web server](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel/host-filtering)
* [Host ASP.NET Core on Linux with Nginx](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/linux-nginx)
* [Configure ASP.NET Core to work with proxy servers and load balancers](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/proxy-load-balancer)
* [systemd execution environment configuration (AmbientCapabilities)](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html#AmbientCapabilities=)
* [Linux capabilities](https://man7.org/linux/man-pages/man7/capabilities.7.html) "CAP_NET_BIND_SERVICE" section