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


### FILTERING CONSIDERATIONS

We’ve been working backwards, starting with the use case of the PDE, then discussing the data contract, and understanding how we get the data contract from the raw data. There is one more important step, which happens at the very start, before the data is first sent into the data primer - this is the data filtering. This involves filtering down the records in the raw data so we have something that makes sense in the context of the PDE and that can be actually used by the PDE. For example, the raw data contains not just maternity benefit applications, but rather *all* EI applications. We are only concerned with the maternity applications, so that is one of our initial filters. We will briefly list and justify these filters. There are some very important consequences of this filtering, which will be discussed in the next section.
EI applications that are not classified as maternity benefits are filtered out
EI applications that were created before 2019 are filtered out. We could leave these in, but we believe that more recent data is going to be more relevant
EI applications that are not filed online are filtered out. This is because there is potential for those to not have the adequate RoE data, which is needed for our calculation
EI applications that are missing important data, such as the postal code, are filtered out. This is data that is required for the average income calculation.
RoE records that are not a “regular” pay period type are filtered out. Regular pay period types include weekly, bi-weekly, semi-monthly, and monthly. These are filtered out because our average income calculation is only able to handle the regular types for now
Instances where an individual has more than one RoE are filtered out. This is also because the current code is only able to handle one RoE per person, though that is an improvement to be made in the near future.
RoEs that have a total insurable hour amount of less than 600 are filtered out, since we know that those would not qualify for maternity. The goal of the PDE is to focus on the entitlement amount, rather than eligibility. Eligibility is a complex determination, but this is a quick check that can filter out some ineligible applications.

This filtering is the very first step taken against the raw data. We filter this data down to some custom tables. These tables are then fed into the data primer, which performs the RoE -> average income calculations, transforms the data into the PDE contract, and stores it in the database to be used by the PDE.

Part III: Data Considerations and Limitations

Now that we’ve discussed the flow from the raw data to the data contract required by the PDE, we will now discuss some of the limitations and special considerations around this flow. We want to be very careful and precise in the way we describe the data, so that it doesn’t create any false impressions. 

The first step we take on the data is to filter it into something more usable, but before that happens, it’s worth reiterating that the data comes to us in a “scrubbed” format - that is, it has been made to be non-identifiable. Things like the postal code and the date of birth have been truncated. There is a careful balance that must be found when working with data. If the data that is used is too precise, then it risks becoming identifiable data. If it is not precise enough, then the models that we generate with our application will become less relevant. Privacy is a top concern, so this truncated data is completely acceptable. This is the format that the data has come to us, so we can feel secure knowing that we can do almost anything with this data and retain appropriate privacy. 

We can now discuss the data filtering. Once again, there is a different balance that must be found when we filter the data. There are two primary reasons for filtering the raw data. The first reason is that it doesn’t make sense in our context. An example of this would be filtering out EI applications that are *not* maternity applications. This could also include filtering out older data, since newer data will likely be more relevant. This type of filtering is both desirable and necessary. The other type of filtering involves filtering out data that doesn’t work within our system. This includes filtering out people that have multiple records of employment and filtering out records with irregular pay periods. The only reason that we perform this filtering is due to shortcomings in the system. Our system is new and as of right now is unable to handle multiple records per person and irregular records. This filtering can be dangerous with a modelling system, since by filtering out that data, we may be ignoring some important demographics, especially if those filtering criteria are “proxies” for other demographic data. For example, suppose having multiple records of employment is a proxy for education level. This would mean that any model where we aggregate the results using the education level would be flawed. We must be very careful with this filtering, and any filters that are present should be adequately communicated to the user.

At this point it is worth reiterating that the PDE is currently in prototype mode. It is constantly being improved on. As the system becomes more sophisticated, we will be able to remove the filters and accommodate more data, making the results more precise. Two of the filters already mentioned have higher priority, particularly the ability to handle multiple records of employment per person. Any improvement on this process will also need to be collaborative with policy experts, to ensure the rule is being captured precisely. While these improvements are being worked on, we still feel that we can show the prototype, while making sure that we acknowledge these limitations and making the potential impacts clear to the users. If a limitation is more complex, then it will be worth studying the exact impact of the filtering. For example, if it proves to be very difficult to handle all of the irregular pay period types, then we may want to do some data analysis to determine more precisely what the impacts of filtering that data are, so that these impacts can be communicated to the user. We also have a separate database that is filled with completely randomized mock data. The data we present in the PDE can easily be swapped out with the mock data if the presentation of the real-yet-filtered data becomes a concern. The content of this mock data can also be exaggerated to highlight the fact that it isn’t real, to further allay any concerns about the perception of using real data (e.g. making up fake names for provinces). 

Conclusion

This document has discussed the goal of the PDE as being a prototype for a data-driven tool, based on a Rules-as-Code engine, that can measure the impact of proposed policy changes to the maternity benefits program. The proposed policy changes are limited to altering the key numbers involved in the maternity benefit calculation. The PDE proceeds by running the original set of rules and the proposed change on a population of individuals and aggregating the results along different demographic properties, such as education level, language, and age. 

The PDE defines a simple data contract for the data that is used in the PDE. Any relevant raw data that we have access to must first go through a ‘priming’ process, which involves transforming the raw data into the data contract. The raw data that we have access to must first be filtered down to something that is appropriate and usable for the PDE. Some of these filters are needed, to ensure the data being used makes sense within the system. Other filters are reflections of current limitations of the program, and will be removed once the program is able to handle them. We want to ensure we capture any consequences and implications of our data-handling process and communicate these appropriately to the users and stakeholders.
