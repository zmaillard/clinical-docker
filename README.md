# Building A .NET Core Web Application and Publishing To Azure Container Registry

The work here was done in a WSL Ubuntu 18.04 environment, but should work in any environment that can the .NET Core SDK can be installed into.

### Prerequisites
- Docker Desktop - https://www.docker.com/products/docker-desktop
- .NET Core SDK (v 3.1 Is Used In This Code) - https://dotnet.microsoft.com/download
- Code Editor of Some Sort.  This was put together in VSCode and vi
- GitHub Account
- Azure Subscription

### Create An Empty .NET Web API Project

Create a new project.  By default the dotnet CLI will name the project the same as the directory it is invoked in.  In this case `clinical-docker`.  The `webapi` project will create a .NET Core WebAPI project with some sample weather endpoints.
```
>  mkdir clinical-docker
>  dotnet new webapi
```

To make sure it worked, run `dotnet run` which will fire up a webserver at `localhost:5000` (by default).  You should be able to verify you are getting some data back at this url: `https://localhost:5001/WeatherForecast/`.  By default it sets the site up to use https with a self-signed certificate.  Your browser will give you all kinds of warnings - just ignore them in this case.

### Create A Docker Container
Create a new file called `Dockerfile`.  You can see the final version of the `Dockerfile` in this repository.

The `Dockerfile` is a manifest/build instruction of how your project will be installed and run within a container.

Line 1 defines the base operating system image this container will be based off.  The .NET Core Dockerfiles are optimized to use two different containers. The first container contains the full .NET SDK which is used to compile the application.  The second container is a small lightweight image that contains the compiled code.  This lightweight image is the one that will actually be run.  The full sdk container will be labled `build-env` and in the image Microsoft provides you will need to supply the version of .NET core you are using.  In this case version 3.1.
```
FROM mcr.microsoft.com/dotnet/core/sdk:3.1 AS build-env
```

Line 2-4 defines a working directory `/app`, not necessary, but keeps the code organized for later steps.  Then the project `csproj` file is copied into the container using the `COPY` command.  Finally, the RUN command will execute a `dotnet restore` on the newly copied csproj file.  `dotnet restore` just installs all of the NuGet files required for your project into the container.
```
WORKDIR /app
COPY *.csproj ./
RUN dotnet restore
```

Line 5-6 compiles the source code.  First the contents of the local directory (and sub-directories) are copied into the container.  This includes all the source code.  The directory structure will be preserved in the container.  Next, a `dotnet publish` command is run to compile the code using a `Release` configuration and outputs the results into the `out` directory.
```
COPY . ./
RUN dotnet publish -c Release -o out
```

Lines 7-10 copies the compiled web application into a new smaller container that does not contain the SDK.  The `FROM` statement pulls down a new base image, again for version 3.1 of the .NET Core runtime.  A working directory is then defined to contain the application.  Finally, the `COPY` command copies the compiled application from the `build-env` container (container defined on Line 1) into the new container.  The artifact is `/app/out` (app being the working directory, and out the name of the output direcory defined from the `dotnet publish` command above).
```
FROM mcr.microsoft.com/dotnet/core/aspnet:3.1
WORKDIR /app
COPY --from=build-env /app/out .
```

Line 11 - Last but not least, an `ENTRYPOINT` is defined to execute the application when the container is launched within docker.  In this case the command is `dotnet clinical-docker.dll`.  There are many ways to launch applications in Docker some more complex than others.
```
ENTRYPOINT ["dotnet", "clinical-docker.dll"]
```

### Build The Container
On the command line run the command:
```
> docker build . -t clinical-app
```

That will generate a new image based on the Dockerfile defined above with the tag `clinical-app`
