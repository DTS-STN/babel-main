# Introduction to the PDE

This page provides the definition and motivation of the Policy Difference Engine and Rules-as-Code approach

## Defining the PDE
The goal of the PDE is to measure the impact of potential changes to rules. By “rules”, we are mainly referring to government rules (legislation, regulation, policy). Team Babel has been working on a prototype for the PDE specifically for Maternity Benefits. Our goal is to show what can be done with a PDE using Maternity Benefits as a proof of concept. It can be assessed as a valuable tool for policy research, expanded to encompass further government rules, and the technology and architecture could be used for related software projects.

We will use the Maternity benefits case as an example to illustrate the function of the PDE. The formula for calculating how much maternity benefits an eligible applicant is entitled to is as follows. It is assumed that the applicant is already deemed to be eligible and that you know the average weekly income (AWI) of the applicant (which is calculated from a record of employment). Let’s suppose an individual has an AWI of $1000.
- Take 55% of the AWI (55% of 1000 is 550)
- Compare that value with a predefined maximum of $595 (550 < 595, so use 550)
- Multiply that number by the number of eligible weeks, which is 15 (550 * 15 = 8250)

Another example, where the applicant has an AWI of 700
- Take 55% (55% of 700 is $385)
- Compare with the maximum (385 < 595, so use 385)
- Multiply by 15 (385 * 15 = 5775) 

The entitlement for the first applicant is $8250 and the second applicant gets $5775. Now suppose someone wants to change the legislation around maternity benefits. There are countless changes that could be made to maternity benefits, but we will restrict our changes to altering the three parameters present in the previous calculation, namely:
- The percentage (currently set at 55)
- The maximum weekly amount (currently at $595)
- The maximum number of weeks of entitlement (currently at 15)

So an example of a change could be the following:
- Increase the percentage to 60%
- Drop the max weekly amount to $500
- Increase the number of weeks to 16

Now let’s look at our applicants with these new parameters. For the AWI of $1000:
- Take 60% of the average weekly income (60% of 1000 is 600)
- Compare that value with 500 (600 > 500, so use 500)
- Multiply by 16 (500 * 16 = 8000)

And for the AWI of $700:
- 60% of 700 is 420
- 420 < 500, so keep 420
- 420 * 16 = 6720

We see that the first individual (With AWI of $1000) loses money as a result of the change, and the second individual (With AWI of $700) gains money. These are just two mock examples of measuring the impact of this change. What would be more valuable is if we had real data on a large number of individuals, so that we could see the larger impact of the change.  Demographic data associated with the individuals could also be used to aggregate results of the proposed change and assess the impact from an equity lens. This would be an example of data-driven policy-making, and this is what the PDE seeks to accomplish. 

## Defining Rules as Code (RaC)
The PDE requires a reliable encoding of the maternity benefits calculation. Without this reliable encoding, the results of the calculations would be incorrect and therefore meaningless. This brings us to one of the main motivators of the PDE - the “Rules as Code” approach. The maternity benefits calculation is part of Canada’s EI act, an important and lengthy piece of legislation. Legislation, regulation, and policy can be thought of as the government “rules” that govern many aspects of our lives. Since these are important rules, it follows that there will be many software applications that need a reliable encoding of these rules, one of which is the PDE (other tools might include HR software, benefits finder applications, service desk tools, etc.). As with all software, it is important that these rules are coded correctly. The specific problem with translating written government rules into code is that the government rules are very complex and often ambiguous. For any given government rule, policy experts will be required to help concretely define the business requirements by clarifying the complexities and resolving ambiguities. Only once this is done can the rules be precisely encoded. 

The fact that these rules are needed across different applications combined with the fact that the business requirements are very complex motivates the idea of a reusable system. Suppose that two different applications require the same government regulation. It would be redundant for both of those teams to spend time reading and interpreting the regulation and then developing and maintaining a coded version of it. What would be far more useful is to develop the system once and then open it up for re-use by other applications that need it. This could be in the form of a software package or exposed as an API. 

Almost any software program is going to have “coded rules”, sometimes called the “logical layer” or the “business layer”. We’ve adopted the name “Rules-as-Code” for this collaborative approach that results in a reusable system. In addition to collaboration and reusability, some other important features include:
Testability: A test-driven approach can be taken to ensure the coded rules are accurate. These tests should also be done collaboratively between technical and policy experts
Transparency: Since these are rules that impact the lives of many people, the code should be made open-source. The rules themselves are public, so the coded version should be as well. Opening the code up can invite others to find bugs in the code or even systemic issues with the interpretation of the rules themselves.
Precision: The Rules-as-Code system should encode precisely the piece of legislation/regulation/policy itself - nothing more, and nothing less. The goal is to be as close to the written document as possible, so that consumers that use the reusable system have clear expectations of what the system is and isn’t capable of. If the rules are exposed as an API, then an Open API Specification can help clarify this for consumers.

## Existing Systems
At the start of the fellowship, one of our first ideas was to scan the existing environment to see what already existed for something that resembled a PDE. We were looking for data-driven simulation engines that could potentially be used to inform policy, in particular those related to benefits and entitlements. We came across a few relevant tools, two of which we will mention here.
LexImpact: This is a web application created for the income tax system in France that allows you to see the impact of proposed changes to the income tax system on a set of customized individuals.
SPSD/M: This is a desktop application created by StatsCanada for measuring the impact of changes to many different government rules. It operates by allowing the user to propose changes to various programs (such as the three mentioned earlier for maternity benefits), and then it runs a simulation of these changes on a population, which is created by carefully merging various anonymous surveys and datasets. It is a very sophisticated piece of software with many use cases. 

The goal of the PDE is not to replace any existing systems or policy analyst positions, but rather to provide an example of a data-driven tool to the policy-making process that is based on a Rules-as-Code system. The work being done by Team Babel involves creating a coded prototype for the PDE.

## PDE Components
Based on our research and existing systems, we’ve identified 4 key components of the PDE, which we will briefly outline here. The technical details of the software components will be expanded upon at the end of this document.


We begin with the rules engine. This is an API that encodes the rules for calculating maternity benefit entitlements for a given person. It takes a “Rules-as-Code” approach, meaning the rules are defined collaboratively alongside policy experts, and the system is exposed as a reusable API. A contract is defined where a consuming application sends the engine details of the individual (such as average income) as well as the rule values to use in the calculation, and the engine returns the amount that the individual is entitled to.

The data represents the different individuals that are being simulated. Each data point contains data that is required for the actual calculation (e.g. average weekly income), as well as demographic data, which can be used for aggregation purposes. 

The simulation engine connects the data with the rules engine. It is responsible for taking each data point (representing a person) and running it through the rules engine, i.e. calculating the entitlement amount for that person. For the purposes of the PDE, it will actually run each person through the rules engine twice: Once for the existing rule (base case) and again for the proposed change (variant case). Once a simulation is complete, the simulation engine is also responsible for storing the results of the simulation in a database, so that they can be accessed and viewed for further analysis. The simulation engine has the functionality to store the data points for subsequent simulations, run the simulations, and fetch the results of a simulation.

Finally we have the front-end. This is a web application where the user interacts with the PDE by proposing policy changes and viewing the results and impact of those changes. The input involves a basic user interface, and the output involves more sophisticated charts and aggregations, which are created using Microsoft PowerBI. The front-end connects to the simulation engine, both when requesting a new simulation, and when viewing the results of the simulation.
