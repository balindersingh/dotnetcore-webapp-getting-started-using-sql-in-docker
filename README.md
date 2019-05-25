# Summary 
* We are going to build a net website using .Net core template on Mac (steps could be same for Linux/Unix)

# Description
* We will be using MSSql 2017 hosted in a docker container.
* MSSql on docker is just for development, for production you can use Azure DB hosted on Azure or simply deploy docker in Azure containers (I never tried that - possible next article :) ).

# Implementation
### As a pre-requisite we will be setting up SQL server using Docker.
### Install docker on your machine if it is not installed.
### Once installed let's test if docker works fine on your machine. To check, open your terminal and run following command
```
docker --version
```
### and you should see its version, like this
*Docker version 18.09.2, build 6247962*
### After installing we need to start docker application before moving forward. You can do this by Docker GUI or by starting it from cli. Following command is tested in Mac only:
```
open --background -a Docker
```
It will take a while to docker to start and be ready to take further commands
## Now as we successfully start docker, Let's get the latest microsoft mssql image into our docker: 2017 sql server
```
docker pull microsoft/mssql-server-linux:2017-latest
```
Screenshot-docker_pull_sqlserver2017
# Let's create a container and execute the container.
## What we are doing
* run: It creates a container and execute it.
* name: an alias for container. we will use it to start and stop the container
* e: it's a KEY=VAL pair to set environment variables
* p: It publish container's port to the host
* d: it runs container in detached mode i.e. in background
```
docker run \
   -e 'ACCEPT_EULA=Y' \
   -e 'MSSQL_SA_PASSWORD=Strong!Passw0rd' \
   -p 1401:1433 \
   --name sql2017 \
   -d microsoft/mssql-server-linux:2017-latest
```
Screenshot-docker_run_sqlserver2017
## Let's set our docker container to interactive mode so that we can run some bash commands like sqlcmd which are available in our sql2017 container
```
sudo docker exec -it sql2017 "bash"
```
Screenshot-docker-exec-bash

#### then it will ask for your machine password. Type the password and hit enter and now you are in interactive mode
#### Now we need to run sql commands. We will use sqlcmd cli. To do that we will create a sql session so that we can fire sql commands
```
/opt/mssql-tools/bin/sqlcmd -S localhost -U SA
```
Screenshot-sql-cmd-prompt
#### Let's create a new Database called MySampleDB. It won't run following query immediatly to run it you have to type 'GO' in next line.  Let's create database and then list all databases exist in current instance of SQL
```
Create Database MySampleDB
SELECT NAME FROM sys.sysdatabases
GO
```
Screenshot-sql-cmd-create-list-DB
> NOTE: you can run any other sql command here. 
### To end your sqlcmd session, type:
```
QUIT
```

## To detach the current Docker interactive mode without exiting the shell,
> use the escape sequence **Ctrl-p** + **Ctrl-q**
## To stop the container when you are not using it run :
```
docker stop sql2017
```

## To restart the image using previous container name 'sql2017'. It will start the container with previous settings
```
docker start sql2017
```


# Reference: 
* Article on how to setup docker for sql: https://database.guide/how-to-install-sql-server-on-a-mac/
* Docker's commands information: https://docs.docker.com/v17.09/compose/reference/
* MS SQL Docker setup and commands:
* https://docs.microsoft.com/en-us/sql/linux/sql-server-linux-configure-docker?view=sql-server-2017
* https://docs.microsoft.com/en-us/sql/tools/sqlcmd-utility?view=sql-server-2017
* Azure Studio (cross-platform SQL Studio with limited capabilites): https://docs.microsoft.com/en-us/sql/azure-data-studio/what-is?view=sql-server-2017

---
# Create a minimal website using template called 'webapp' and additionally specifing addon using --auth
```
dotnet new webapp --auth Individual -o WebApp1
```
# Create a database in docker or in Azure and get the connection string
```
Server=127.0.0.1,1401;Database=MySampleDB;User Id=SA;Password=Strong!Passw0rd
```
### The default sql server for this git repo was SQLite but I wanted to use full SQL server hosted on my Docker image So made follwing changes in 2 files
### In appsettings.json added my connection string 
```
// FROM 
"DefaultConnection": "DataSource=app.db"
// TO 
"DefaultConnection": "Server=127.0.0.1,1401;Database=MySiteDB;User Id=SA;Password=MyStrongPassword"  
```
### Startup.cs
```
// FROM
UseSqlite
// TO
UseSqlServer
```
### *_CreateIdentitySchema.cs change
```
// FROM 
SQLite:Autoincrement
// TO
Autoincrement
```
# Updating/seeding database with initial setup
```
dotnet ef database update
```
### At this point DB Table should be created in your SQL DB
### Now let's build the solution and run it.
```
dotnet build
dotnet run
```
### It should show you where website is hosted locally on your machine. You should be able to go to that localhost (ignore https warning and proceed).
# All next steps are optional, if you want to add more functionality For example : a class/table i.e. Project and its scaffolded views etc.
```
dotnet add package Microsoft.VisualStudio.Web.CodeGeneration.Design
dotnet aspnet-codegenerator identity -dc WebApp1.Data.ApplicationDbContext --files "Account.Register;Account.Login;Account.Logout"
```
## You might get 'No executable found matching command "dotnet-aspnet-codegenerator"' error when you ran last command . It means you might be missing dotnetcore code generation tool on your machine. Before netcore 2.1 release it used to be a local nuget package reference but with 2.1 release it has to be installed gloabblly. To fix that run following command and it will install dotnet code generation tool on your machine globally.
```
dotnet tool install --global dotnet-aspnet-codegenerator
```
Screenshot-dotnet-apnet-generator

---
# Now let's Create a new model class i.e. Project.cs in Models folder. See example in refrenced MS Doc (dotnet model)
# Create one more class i.e. ProjectDBContext.cs in 'Data' folder. See example in refrenced MS Doc (dotnet model)
# Now we have the Model class we can scaffold controller actions and CRUD views under Views folder. To do that we will use 'aspnet-codegenerator'. 
```
dotnet aspnet-codegenerator controller -name ProjectsController -m Project -dc ProjectDBContext --relativeFolderPath Controllers --useDefaultLayout --referenceScriptLibraries
```
### Make sure you have it installed already (above 2.1 you need to install it globallly on your local machine before 2.1 dotnet core it can be installed locally to a project)

# Now in StartUp.cs -> ConfigureServices method add following line after the existing AddDbContext command
```
services.AddDbContext<ProjectDBContext>(options =>
                options.UseSqlServer(
                    Configuration.GetConnectionString("DefaultConnection")));
```
# Now run migrations for new model
### If you already have more then 1 context then you need to pass --context <yournewmodelcontext>. the one you added in ConfigureServices
```
dotnet ef migrations add InitialProjectModelCreate --context ProjectDBContext
```
### You will notice a new folder name called 'Migrations' with some files.
### Above command basically generates a script to create the initial database schema. The database schema is based on the model specified in the ProjectDBContext class (in the Data/ProjectDBContext.cs file). 
# Now let's update the DB with the changes we intend to migrate with the above migration
```
dotnet ef database update  --context ProjectDBContext
```
## At this point DB Table should be created in your SQL DB
# Now let's build the solution and run it
```
dotnet build
dotnet run
```
## It should show you a link with port number where current site is hosted on your localhost

# How to add new fields into existing table/model class
## Simply add properties with Data Annotations in your Model/Project.cs class as mentioned in refrenced link (MS Doc below). Then run the following commands:
```
dotnet ef migrations add InitialProjectDBModelUpdate --context ProjectDBContext
dotnet ef database update  --context ProjectDBContext
```


# References:
* dotnet core authroization : https://docs.microsoft.com/en-us/aspnet/core/security/authorization/simple?view=aspnetcore-2.2
* dotnet model: https://docs.microsoft.com/en-us/aspnet/core/tutorials/first-mvc-app/adding-model?view=aspnetcore-2.2&tabs=visual-studio-code
* How to add a new model: https://docs.microsoft.com/en-us/aspnet/core/tutorials/first-mvc-app/adding-model?view=aspnetcore-2.2&tabs=visual-studio
* How to add new field to your existing model: https://docs.microsoft.com/en-us/aspnet/core/tutorials/first-mvc-app/new-field?view=aspnetcore-2.2&tabs=visual-studio