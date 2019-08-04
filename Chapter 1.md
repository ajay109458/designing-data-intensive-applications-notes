
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

#### Human Errors

Human -> Build and Operate software. Even if they have best intension human are known as unreliable. 

A study pointed out that configuration by operator leads most of outages. 

Steps to design reliable system in spite of unreliable human
- Design system in a way that minimizes the opportunities for the error. E.g.
  - Well-designed abstraction
  - API 
  - Make it easy to do right thing and discourge wrong things. 
- Test throughly 
  - Unit tests
  - Whole system integration tests
  - Manual testing
- Allow quick and easy recovery from the human errors 
  - Like rollback
- Setup detailed and clear monitoring such as performance metrics error rates.


### Scalability
Scalability is ability of the system to cope up with increased load.

Load can be described with a few numbers which we call as `load parameters` 

#### Tweeter example

**Post Tweet**
A user can publish a new message to their followers ( Average -  `4.6k requests/sec` , At Peak - `12k requests/sec` )

**Home Timeline**   
A user can view the tweets posted by the people they follow ( `300k requests/sec` )

Handling 12K write requests per second is easy. Twitter's scaling challenge is not primarily due to tweet volume but due to `fan-out` - each user can follow many people and each user is followed by many people. 

There are following ways to deal with this operation    
1. Posting a tweet simply insert a new tweet into global table of tweets. When a user requests their home timeline, look up all the people they follow find all the tweets for each of the user. Merge them and sort by time. 

```
SELECT tweets.*, users.* FROM tweets
  JOIN users ON tweets.sender_Id = users.Id
  JOIN follows ON follows.followee_Id = users.Id
  WHERE follows.follower_id = current_user
```
2. Maintain a cache for each user timeline. When a user post a tweet, lookup all the user who follow that user and insert the new tweet into the user timeline record. 

Earlier twitter used approch #1 but later shifted to the approch #2 becacuse magnitude of reading timeline record is much higher then the posting tweet.     

Now, disadvantage with approch #2 is that posting a tweet will do a lot of extra work. On average tweet is delivered to about 75 followers. So `4.6K tweets` now will result into `345k` write operation to the hometimeline cache.

We took avg but some user might have over 30 millions of the follower. It means single tweet from such a user will result into the `30 millions` writes to home timeline. Tweeter tries to deliver a tweet within 5 second and doing this for `30 millions` write is a significant challenge. 

So, now twitter used mixed approch - Approch #2 for all users and Approch #1 for celebrity.

### Maintainability

Maintainance of a software includes 
- Fixing bugs
- Keeping system operational
- Investigating failures
- Adapting to new platforms
- Modifying it for new use cases

We should consider following aspects while designing 
- Operability - Easy for opeartion team to keep system running.
- Simplicity - Making it easy for new engineers to understand the system, by removing as much complexity as possible from the system.
- Evolvability - Make it easy for engineers to make changes to the system in the future, adapting for unanticipated use cases as requirements change.It is also known as extensibility. 
