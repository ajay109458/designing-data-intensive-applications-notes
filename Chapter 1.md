
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


