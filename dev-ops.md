# PDE Dev Ops
This document will discuss the DevOps aspects of PDE, including what needs to be done to get the PDE running from the ground up.

## Overview
All of the services are currently hosted on Azure. The following is a list of the main services
- Azure Container Registry (acr): All of the app services are deployed using docker images which are hosted here
- Rules Engine: .NET Web API hosted on Azure App Service using a docker image from the acr
- Simulation Engine: .NET Web API hosted on Azure App Service using a docker image from the acr. Interacts with the production server/database.
- Babel Web app: .NET MVC Web Application hosted on Azure App Service using a docker image from the acr. Interacts with the Simulation engine.
- Server/Database: Azure SQL Server - Holds the data and results associated with simulations run by the Simulation Engine

Note: You may want to have multiple deployments of certain components to accommodate separate datasets. For example, you may want to have a system that uses real data, and another that uses mock data. This will require separate services. For example, you would likely need to create two separate server/dbs, two separate simulation engine deployments (each one connecting to a different server/db), and two separate web app deployments (each one connecting to a different simulation engine). You may not need a separate deployment for the rules engine, since it is not associated with any data - it simply acts as a rules calculator.

### Optional Services
- Rules Gateway: Azure API Management layer. Re-routes traffic to the Rules API
- OpenFisca Engine: Optional component of the rules engine. Exposed as a Web API, and hosted on Azure App Service using a docker image from the acr
- PowerBI: Allows sophisticated visualizations of the simulation results

## Starting from Scratch
Suppose you have access to all of the [PDE repos](https://github.com/DTS-STN/babel-main/blob/main/components.md), but nothing is deployed to any environment. You can start by running the projects locally to ensure they are working together.

### Software and Services
The following software/services are needed for basic development
- [.NET Core 3.1 SDK](https://dotnet.microsoft.com/download/dotnet/3.1)
- [Postman](https://www.postman.com/)
- An IDE, such as [VS Code](https://code.visualstudio.com/)
- [Azure subscription](https://azure.microsoft.com/en-ca/)

If working with the data primer and doing data transfers, you will also need:
- A SQL client, such as [SSMS](https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms?view=sql-server-ver15)
- Access to the Government Data lake
- Oracle SQL Developer (for interacting with the data lake)

### Running Locally
Pull down the three main repos:
- [babel-rules-engine](https://github.com/DTS-STN/babel-rules-engine)
- [babel-simulation-engine](https://github.com/DTS-STN/babel-simulation-engine)
- [babel-web-app](https://github.com/DTS-STN/babel-web-app)

All three projects are built using .NET Core 3.1. They all run as web applications, so they will need to be running on different ports when developing locally. This can be configured in the launchsettings.json file. By default, the rules engine is on 6000/6001, the simulation engine is on 7000/7001, and the web app is on 5000/5001.

Let's start with the Rules Engine. If you aren't using OpenFisca, then there are no configurations that must be set. Simply run the application. 

Next is the simulation engine. If you simply want to run it for development and testing purposes, then you may not need an actual database, since there is a built-in cache storage implementation. Ensure the cache storage is being injected in and that the mocks will be generated on startup. Both of these can be confirmed and configured in the Startup.cs file. If you want persistent storage and need a database, you can spin up an Azure SQL database. See the section on setting up a database for this step. Once the database is set up, ensure the connection string is set appropriately in appsettings (or environment variable), and the the DB storage systems are being injected into the applications (Startup.cs). You will also need to configure the Rules API Url in the settings. If the Rules API is running on `http://localhost:6000`, then set that as the Rules Url. Run the simulation engine.

The projects are set up to read the appsettings.[Environment].json files, or if those are missing, then it looks for environment variables. When running locally, you can enter the config settings in the appsettings.Development.json. Take care to not commit these to source control. They are not needed for deployment. 

For the Rules and Simulation Engines, there are postman collections in the repos which can be used to test the projects locally (and also when they are deployed). Import these collections into Postman and create separate environments for the rules and simulation engines.

You may encounter an SSL connection error when running the application - for example when the Simulation API attempts to communicate with the Rules API. Since we're developing locally, you can run this command to get rid of the error: `dotnet dev-certs https --trust`. [Relevant Link](https://stackoverflow.com/questions/52939211/the-ssl-connection-could-not-be-established)

Finally, we need to run the web app. The only config that needs to be set is the Simulation Url. Set that and run the project. You should be able to open it up in a browser, run a simulation, and view the results.

### Setting up a Database
This is probably the first service you will want to set up on Azure. 
- Log in to Azure
- Create a new SQL Server. Set the name, location (in Canada), and credentials as desired.
- Ensure that you 'Allow Azure services and resources to access this server'
- Finish creating

Next you will need to create a database on the server
- Create SQL Database
- Enter the server that you just created as the server
- Choose an appropriate pricing tier. Basic DTU-based is a cheap option.
- Other default values are fine
- In the networking tab, ensure that you 'Add current Client IP address'. This will allow you to connect locally
- Finish creating

Once the server and database are created, verify in the server firewall rules that your client IP is added and that the 'Allow Azure services and resources to access this server' is checked.

If you are deploying both a prod and a mock database, then you will perform the server and database creation twice, so you have two server/databases with separate credentials. You will need the connection string for the database when running the simulation engine.

### Setting up Azure Container Registry
The three main projects have Dockerfiles that can be built and pushed to a container registry for deployment. Before doing any deployments, we must first create the registry.  Simply create a Container Registry with your desired name and other options. Then find the Access Keys in the service settings and grab the username and password. You will need them for the next step. Also note the name of the acr server (e.g. something.azurecr.io)

### Github Deployment Workflow

Before creating an app service, we must first push an initial image to the acr for each app service we want to deploy. This can be done manually from the command line with docker commands, or it can be done using a github workflow. We will walk through the github example. 
- Go to the github repo of the project you want to deploy. 
- Go into the Settings > Secrets, and create two new secrets. Call them something like ACR_MY_ENV_USERNAME and ACR_MY_ENV_PASSWORD. Copy the credentials from your ACR access keys in there.
- Go back to the repo files and navigate to the workflows folder
- Copy the contents of a 'deploy' file to a new file in the workflows folder. Name it something intuitive, e.g. deploy-my-env.yml
- Inside your new file, make the following changes
  - Change the 'ACR_SERVER' env to your acr server (e.g. something.azurecr.io)
  - Change the 'APP_NAME' env to whatever you want the image to be called (e.g. mysimulationapi)
  - Change the username and password to be the github secrets you just created
- Save and commit the file

The workflow is set up to be deployable by clicking. Navigate to the github repo's 'Actions' tab, find your workflow, and run it. Navigate back to your acr service in Azure, then look at the repositories in the service, and your image should be there. 

Perform this step for each service you want to deploy. If you are deploying mocks, you will need separate ACR repositories and separate deployment yaml files in the workflow.


### Creating an App Service
We can create app services for each application we want to deploy. You can deploy as few as 3 (rules engine, simulation engine, web app), but if you want mocks then you will be deploying 5: rules engine, simulation engine x2, web app x2. You can also deploy OpenFisca in the same way. 
- Create an Azure App Service
- Choose a resource group, a name, and region
- For the Publish option, choose **Docker**. For the Operating System, choose **Linux**
- Select the pricing tier. As of writing there is a free option
- In the Docker tab:  
  - Options: 'Single Container'
  - Image Source: Azure Container Registry
  - Image: (Find the image name you created in the previous step, e.g. mysimulationapi)
  - Tag: latest
- Review and Create

Now we just need to set up the config settings. We did not commit the appSettings.Production.json file to source control, so instead we are going to inject environment variables through Azure. Navigate to your newly created app service, find the configurations (usually in Startup.cs), and set the appropriate config variables. Saving this will restart the application.

