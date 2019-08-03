
## Basics

Many application today are `data-intensive` instead of `compute-intensive`. CPU power is rarely a limiting factor.

Data-Intensive application built from standard building block
- Store data so that same or another application can use it later ( `database` )
- Remember the expensive operations, to speed up read ( `cache` )
- Allow user to search data by keyword ( `search index` )
- Send a message to another application, to be handle async ( `stream processing` )
- Periodically crunch large amount of data ( `batch processing` )

## Definitions

#### Reliability
The system should continue to work correctly (at desired level of performance ) even in face of adversity 
(Hardware failure, software failure or even human failure )

#### Scalability
As the system grows ( in terms of traffic volume, data volume or complexity), there should be reasonable ways to deal with that growth.

#### Maintainability
Over time, many different people will work on a system, they should be able to work productively. 

## Reliability

#### Reliability meaning
- The application should work as user expected
- It can tolerate user making mistakes and using software in unexpected ways. 
- Its performance should be good in required use case, under expected load and data volume. 

If all above things means `working correctly` then `reliablity` is `continue to work correctly even things go wrong.`.


- Things that go wrong are known as `faults` 
- A system that anticipate the `faults` and cope up with them is known as `fault tolerant` or `resilient`.

Though `fault-tolerant` term is slighty misleading because its not feasible to make a system that can cope with evey possible kinds of fault. E.g. The entire planet earth got distroyed. So we should mention certain type of faults.

- A fault is being defined as a component of the system is being deviated from its spec. 
- A failure is when system as a whole stop working. 

Its impossible to reduce the probability of faults to zero. So, we should design a best `fault-tolerant system` that prevents
faults from causing failure.

Netflix `chaos Monkey` deliberately induce the faults in the system by randomly killing individual process to ensure fault-tolerant system.

#### Hardware faults

Few hardware failures 
- Hard disk crashed
- RAM become faulty
- Power grid has a blackout
- Someone unplug wrong network cable.

Hard disk are reported to have meantime to failure (MTTF) is about 10 to 50 years. Thus on a storage cluster with 10000 hard disks, we can expect one Harddisk will die each day. 

Possible solutions :
- Add redundancy to the individual compoents in order to reduce the failure rate of the system. 
- Disks must be setup in a RAID configuration. 
- Server with dual power supply
- Hot swappable CPU's
- Generator for the power backup.

- If you go with cloud platform, all above mentioned issues are automatically handled from the cloud service.

#### Software failures

Hardware failure are generally random and independent of each other. It means one machine's hard-disk failing doesn
t imply that another machine's disk is going to fail.

- Software failures are harder to antipate and are correlated across the nodes.
- The bug that cause these king of software failure often lie in dormant state for a long time and triggered by unusual circumstances.

There is no quick solution to these types of problem but can take prevention steps 
- Careful think about the assumption and interactions in the sytem 
- Thorough testing 
- Allow individual process to creash and restart
- Measuring, monitoring and analysing system behavior in the production. 



