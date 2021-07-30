# PDE Data Flows

This document discusses how data flows through the system, beginning with the raw data, until it is presented in a dashboard resulting from a simulation. It also discusses some important considerations resulting from this flow.


## Data Contract
There is a considerable amount of work that must be done before the data is actually ready to be used by the PDE, and it is arguably more complicated than the flow of the PDE itself. We can start by defining the data contract required by the PDE, which largely stems from the maternity benefit rule. The only major input required for the maternity benefit calculation is the average weekly income. The only other properties are the demographic properties on which we want to do results aggregation. For example, if we run a simulation and find that 80% of our applicants gain money from the simulation, then that may be seen as a positive thing, depending on policy goals. However, if we zoom in on certain demographics, we may find that some groups are adversely affected. Suppose we look at women between the ages of 30-35 in Nova Scotia, and we see that 95% of individuals in this group actually lose money. So while overall the outcome may seem positive, there may be some hidden results that make the policy change inequitable. This is part of the goal of the PDE and it requires demographic data. For our prototype, we’ve selected a few demographic indicators to illustrate this concept - namely province, age, spoken language (french/english), and education level. We add these pieces of data to the average income to get our data contract for the PDE. Once we have data in this form, regardless of its origin, we can run it against the PDE.

Data contract of the PDE
- Average Weekly Income
- Province
- Age
- Spoken Language
- Education Level

The problem is that the raw, available data will likely not come to us in this form. It may come in the form of a huge dataset that has many relational tables to search through, and it may require some complex calculations. The current situation is that we have access to records of employment and EI application data. Together, this data can be converted into the format specified by our data contract, but it is a very tedious process. The raw data is found in the form of views in a data lake that has been populated with non-identifiable data. There are two primary databases that are being used - the NEIW and the ROE. 

The NEIW database contains the EI application data as well as the associated individual client data. It is worth emphasizing that the client data has been made to be non-identifiable by omitting and truncating certain information. This is how the data comes to us - we do not have any access to the original identifiable data. The ROE database contains the record of employment data, which is used to calculate the average income. This can be linked to the application and client data using a “person ID”. The data that is needed or the contract is spread across a few different tables in these two databases.  We will now walk through the steps taken with this data to get it ready for use by PDE.

## Filtering and Joining the Raw Data
Our first step with this raw data is to filter it down into data that makes sense to use in the PDE - note that this will not make it ready for actual use by the PDE - that will be a separate step. This first step involves eliminating certain data points that either do not make sense for our system or that are not yet compatible with our system. For example, we only want maternity benefit applications, so we need to filter out all EI applications that are not maternity. We also want newer data, so we filter out data that is older than 2019. This initial filtering process doesn’t do any “transformations” on the raw data, rather it filters and joins the data into two new tables. One of these tables (client table) contains the EI application data, demographic data, and some record of employment data. The other (earnings table) contains the associated record of employment pay periods. A record of employment will have many pay periods, which are used for the average income calculation. There is a relationship between these two tables on the RoE ID. The client table contains a RoE ID which links to many pay periods on the earnings table. 

The following diagram illustrates this first step and shows some of the important columns involved. This step is currently done using a set of SQL queries. Once the two tables are populated, they are exported from the data lake and imported into a new database. 



## Data Priming
With these two custom tables, we have something a little bit closer to the required data contract. We have some of the demographic information, but we are still missing the average weekly income. To this end, we need to code the complex calculation that yields the average weekly income from the record of employment. The idea is that we want to take what is stored in our two custom tables, and output data that fits the required data contract. We will then use this on the PDE. A special purpose console application called the ‘Data Primer’ has been written to achieve this. 

The Data Primer first reaches into the database where the two custom tables are stored and loads all of the data. It must then calculate the average weekly income from the record of employment. The code to perform this complex calculation is stored in the RaC engine, which is exposed as an API. So for each data point that comes in, the Data Primer packages up the Record of Employment and client information, and makes an API request to the rules engine, which will calculate and return the average weekly income. The Data Primer will then fill in the other requirements for the data contract (i.e. the demographic data) using other pieces of data from the custom tables. It will build up a collection of “persons” that all fit the data contract. Once it is done processing all of these, it needs to store these objects, so that it can be used by the simulation engine. This storage step is done by calling the simulation engine, which is also exposed as an API. The Data Primer passes in all of the formatted individuals to the API call, and the simulation engine will accept the request and store those values in a database. The data is now primed. When someone requests a simulation to be run from the Simulation API, it will make use of this stored data. The following diagram illustrates the functionality of the data primer:



## Running Simulations
Once the data has been primed and stored in the database by the simulation engine via the Data Primer, new simulations can now be requested from the simulation engine. To create a new simulation, we make another API request to the simulation engine. This can be done manually through an application such as Postman, but more realistically it is done through a user-facing web application (the front-end). Since the data is already loaded in the simulation engine, we don’t actually need to send any data for the simulated individuals. What we do need to send are the simulation cases that we want to run. In the introduction, we went over two possible scenarios for the maternity benefits calculation - a base case and a variant case. Here are the rule parameters we used in each case:

TABLE GOES HERE

Parameter
Base Case
Variant Case
Percentage
55
60
Max weekly amount
$595
$500
Number of weeks
15
16


This is the information that we must communicate to the simulation engine, and so this is what must be included in the API request. When we make the request, the simulation engine will load all of the individuals from the simulation database, and apply both the base case and the variant case to each. The application of the rule is done using the rules engine. So for each simulated individual, two calls would be made to the rules engine - one for the base case and one for the variant case. This call is done using an API call to the rules engine, where we pass in the rule to be used (base/variant case) and the individual data, which adheres to our data contract. In reality, this means a lot of network requests between the APIs, which slows down the process. To optimize the simulation, we’ve introduced techniques such as batch calls and caching. The following diagram illustrates this flow:



Once the simulation has completed, the user is able to request the results of the simulation. These results are then fetched from the DB and sent back to the front-end for visualization and aggregation.
