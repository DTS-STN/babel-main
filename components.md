# Policy Difference Engine

This document gives a higher-level overview of the software components involved in the PDE - please refer to the linked repos and readmes for more detailed information.

## Main Software Components

### Data Primer
[Repo](https://github.com/DTS-STN/babel-data-primer)

This is a console application written in C#. It connects to the database that stores the custom tables from the data filtering step. Note that the data from those tables has been exported from the raw data lake. In order to get access to the data lake, you will need to go through the Oracle Client on the VPN and get appropriate permissions. Run the filtering queries to create the two tables, then export and import those tables into the database. The Data Primer will connect to that database via connection string and look for those specific tables. 

It also has a connection to the Rules engine and Simulation engine using the API endpoints. These, as well as the db connection string, are defined in the config files, and can be updated to suit environment needs (e.g. mock data vs production data). Since it is a console application, it is currently not deployed to any environment and can just be run on a local machine. Ideally, it only needs to be run on rare occasions, such as when we want to update the data that is used for the simulations. If you are fetching completely new data from the data lake, then you can also run the SQL filtering/joining step to create the custom tables and then re-import that for use by the Data Primer. 


### Rules API
[Repo](https://github.com/DTS-STN/babel-rules-engine)

The Rules API is a Rules-as-Code approach to encoding government rules. The current rules engine contains only the code for the maternity benefits calculation, but the architecture is designed to be extendible to other government rules as well. This is a typical Web API, written in C#, that exposes endpoints that trigger functionality related to the calculation of maternity benefits. There are actually 3 endpoints that are used for the full calculation:
- Number of Best Weeks: This call takes in a postal code and returns the number of best weeks, which is used for calculation of the average weekly income. This call is currently used in the processing step of the Data Primer.
- Average Weekly Income: This is the most complex call in the entire PDE system so far. This involves taking in record of employment information, and calculating the average weekly income for an individual, which is part of the required data contract. The calculation has many edge cases and complexities. It requires ongoing consultation with policy experts, as well as thorough unit tests. This call will likely be inaccurate for many record of employment scenarios, but we are taking an iterative approach to improving it as we consult with policy experts. This call is currently also used only by the Data Primer.
- Maternity Benefit Entitlement: This is the main call used by the simulation engine. Once we have data in the format of the required contract, the simulation engine can call this endpoint with the rule parameters and the average weekly income to get the individual’s entitlement amounts for the base case and the variant case. 

The Rules API is hosted in Azure and contains a dockerfile for deployment via the container registry. The API does not store any of the simulation data - it’s role is to act as the calculator for data and rules that get passed to it.


### Simulation Engine
[Repo](https://github.com/DTS-STN/babel-simulation-engine)

This is another C# Web API and is the main “control centre” for the PDE. When a request for a simulation is made, it brings together the data (stored in the simulation DB or cached) and the rules (by connecting to the Rules Engine). Conceptually it is fairly straightforward. It allows for the following main calls:
- Add Persons: This function allows you to insert a batch of persons to be used by subsequent simulations. This call is currently used by the Data Primer. Note that this endpoint is password-protected.
- Create Simulation: This function allows you to create a maternity benefits simulation by specifying the base case and the variant case. Once the simulation is complete, the results are stored, and the engine returns a GUID, which uniquely identifies the new simulation. This call is used by the front-end web application, which is operated by the end user.
- Get Simulation Results: This function allows you to get the stored results of the simulation. It contains the demographic info of each simulated person as well as their base and variant amounts. This call is used by the front-end web application for displaying results of a simulation.

There are a few other calls for managing the simulations. These can be found in the OpenAPI spec of the project.

Like the Rules API, the Simulation API is hosted in an Azure App service and can be deployed using docker. Unlike the Rules API, there are may be multiple deployments of the simulation engine - e.g. one for mock data (which points to a database that is for storing randomly generated individuals), and one for the real data (which points to the database populated via the Data Primer). The multiple environments can be used for testing and demo purposes.  

### Front-end Web Application
[Repo](https://github.com/DTS-STN/babel-web-app)

This is a thin web application that serves as the user interface of the PDE. The only connection it has to the other software components is the simulation engine. Like the simulation engine, there may be multiple deployments to accommodate separate data sets. These deployments would differ only by config settings, such as the Simulation API url.

In addition to a welcome page, the front-end has two main interactive pages. The first is the form for creating a new simulation. This page simply allows the user to enter values for the three rule parameters (percentage, max weekly amount, number of weeks). From this page the user can run the simulation, at which point a call to the simulation engine is made. Once that call is completed, the web app is redirected to the results page, which makes another call to the simulation engine to get the stored results. The page displays useful aggregations of the results. 

The front-end application is written in ASP.NET Core MVC, which uses C# and Razor markup. It is purposefully a fairly slim application, relying on the simulation engine to do all of the processing (which in turn relies on the rules engine). The deployments are hosted in Azure App Services and can be deployed using docker. 


## Working with the PDE
Depending on the type of work you want to do on the PDE, you will need to pull down several repos and get them running locally. You can swap out connection strings and API URLs to customize the environments you are working in. Instructions for running and deploying the software components are found in the readme files in source control. You can also visit the [devops/deployment](https://github.com/DTS-STN/babel-main/blob/main/dev-ops.md) page for more info

The diagram below illustrates how the software components are connected to each other.

![Software Components](https://github.com/DTS-STN/babel-main/blob/main/images/components.jpg)

-----------

## Optional Components

There are a few optional components that have been shown as proof of concepts that can enhance the PDE.

### OpenFisca
[Repo](https://github.com/DTS-STN/babel-openfisca-engine)

[OpenFisca](https://openfisca.org/en/) is an open-source software that specializes in encoding benefits and eligibility systems. It was a system that was being used in pilot projects within DECD prior to Team Babel. For our project, we continued working on experimentation with the technology. We currently have an OpenFisca API running behind the scenes, with the maternity benefits entitlement calculation encoded. For that particular call, the Rules API connects to the OpenFisca API, which performs the calculation, and sends the results back to the Rules API, which would then forward it on to the Simulation Engine. This step is not required and is currently inactive. The entitlement calculation is relatively straightforward, so a version exists in the Rules API code. The connection to OpenFisca can easily be reactivated. The OpenFisca library is written in Python.

### API Management layer
The final component of the Rules Engine is the API Management layer, sometimes called the gateway. This is a layer that sits on top of the Rules API and performs the appropriate routing. The management layer makes more sense in the larger context of RaC. Currently we only have the one Rules API, but if we were to build multiple APIs that handle more government rules, then it would be valuable to have a gateway sitting in front of these APIs to handle cross-cutting concerns, such as routing, logging, error-handling, etc. Both the simulation engine and the Data Primer connect to the Rules Engine using the gateway. Functionally, this just means they connect to a gateway endpoint, which gets re-routed to the Rules API. At this point, the step could be skipped, and the simulation engine and Data Primer could simply connect directly to the Rules API. The API Management Layer is an Azure service that comes with a set of other useful functionality. It allows you to specify an Open API contract for the APIs behind it so that the contract can be made available to potential consumers. It also automatically generates a developer portal that potential consumers can go to learn about the different APIs behind the gateway. There is more potential functionality that can be added to this layer, but for now it is fairly minimal since we are just using the one API.


### Power BI
Microsoft PowerBI is a visualization tool. We have been experimenting with it for displaying useful visualizations and aggregations from the results of our simulations. 

