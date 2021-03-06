# Transactions

In this article we talk about transactions from many aspects. We discuss:
1) ACID, the transaction isolation levels.
2) Various transaction models
3) Pessimistic & Optimistic locking
4) XA Trasnactions
5) Common Transactional Design Pattens
6) Eventual Consistency.
7) How Databases such as MongoDB achieve consistency of data
8) CAP Theorm

## ACID, the transaction isolation levels
### Atomicity: 
Changes are either Committed or Rolled back. We don’t have a superposition of states.

### Consistency: 
Every system has business rules. Before and after the changes the system should be consistent with these Business Rules. Though not necessirily the case during a transaction.

### Isolation:
In a ball dance, you probably don’t want to be stepped on your shoe by someone else. Like wise you don’t want anyone to fiddle with your data when you are updating a row in a RDBMS. Isolation is at your rescue. 
There are primarily 4 Isolation Levels for achieving concurrency:
#### Read Uncommited: 
This is the lowest level of Isolation. Also called Dirty Read, one can read uncommited data. Generally not recommended in multithreaded environment.
#### Read Committed: 
One way to solve the Dirty Read problem is to switch to the next level of Isolation or Read Committed. With this Isolation level, only committed data from other transactions will be visible. Here again you might be surprised with a collegues commited data during your transaction.
#### Repeatable Read: 
If Read Committed Isolation level is not good enough then the next higher level of isolation is the Repeatable Read. Multiple reads are guaranteed to return the same result during every reads until the transaction is active whether or not the other transacations are committed or not.
#### Serializable: 
Though Read Committed or Repeatable Read is practically good enough for most applications, it does come with a trade off of what is know as ‘Phantom Reads’. Within the course of a transaction, two selects would return different number of records. To avoid this one may have to set isolation to Serializable a.k.a Table Level Lock. This level guarantees the table itself will not be touched during an active transaction. This is the higest level of Isolation. Summary: There is however a relation between Consistency, Isolation and Concurrency. Though higher levels of Isolation guarantees better Consistency it does come with a trade off of Lower Concurrency in other words Lower Performance. It is generally recommended to use one of these Isolation Level as much as possible, infact few databases does only support these two levels of Isolation. While you could indicate from your code the level of isolation, do note that your underlying datastore must support that level inorder to work as expected.

### Durability: Changes are persistent or permenant once committed.

## Various transaction models: 
There are 3 types of Transaction Models.
### Local Transaction Model: 
As the name suggests we rather deal with the actual Resource via Local Resource Manager. In case of a Database the autocommit flag to false before being able to preform a commit or rollback of the transaction to achieve the desired effect. Using this technique we can control multiple database calls (from the connection) from within one transaction. 
Cons: Along with a need for a lot of boiler plate code, Local transaction model is not very good at coordinating between multiple resource using XA (Extended Architecture) global transaction. Local transactions require more coding and is prone to errors unlike in Declarative Transaction where most of the code are managed as seperate Aspects.

### Programmatic Transaction Model: 
Programmatic Transaction Model deals with Transaction artifacts rather than the resources directly. In EJB Programmatic Transactions are attainted by virtue of JTA (Java Transaction API) Transaction API. Every JEE Compliant App Server will provide its own implementation of JTA also known as JTS (Java Transaction Service). There are also standalone implementations for JTS like JOTM. With Programmatic Transactions one deals with javax.transaction.UserTransaction instances. In a JEE Environment it can be looked up from a JNDI InitialContext or injected using a @Resource defined by JSR 250 also called Common Annotations. Common operations include begin(), commit(), rollback(), getStatus() etc. Calling begin() associates a new Transaction with the current active thread, calling begin() when the transaction is already active throws a NotSupportedException. 
In an XA Transaction commit() could throw HeuristicException (more on the Exception later). Additionally, with EJB one must be careful to either commit() or rollback() before the method execution completes. It is a common mistake to catch an exception and forget to rollback the transaction. Method execution would throw an Exception in such cases.     
With Spring one could inject a TransactionTemplate to achieve Programmatic Transaction Model. Template along side TransactionCallback instances allows to wrap the business logic within the Transaction Model. Another way to achieve the same effect is using a PlatformTransactionManager.  
In the context of EJB Programmatic Transaction is also called Bean Managed Transaction or BMT. A Transaction cannot be propogated to a BMT from either a calling CMT or BMT, though a CMT can use the transaction initiated by a calling CMT or BMT.  
Declarative Transaction Model: With this model, the begining, committing or rolling back a transaction is the job of the container. Transaction Interceptors wrap the Bean instances performing these tasks. With EJB For a Transaction Rollback, one only needs to indicate or mark a transaction for rollback with EJB by calling the setRollBackOnly() on the EJBContext. If a method throws a RuntimeException then the interceptor performs a rollback on the transaction. Additionally one can throw an Exception marked @ApplicationException(rollback=true) for a checked exception (Not RemoteException/EJBException) to be able rollback a transaction.
DeclarativeTransactions in EJB is defined using a @TransactionManagement as Container Managed. It also allows for different 
#### TransactionAttributes. 
Required
RequiresNew
Mandatory
Never
Supported 
NotSupported 

With Spring 'Nested Transaction' is additionally supported. With EJB the transaction is suspended in the scenario. TransactionManger Interface also supports suspend() and resume() methods. For example if the Bean calls a piece of code which invokes Stored Procedure (which could have DDL which is not executed within the context of a transaction), its best to temporarily suspend the transaction and later resume it after execution of Stored Proc.Spring makes use of TransactionProxyFactoryBean as its approach to Declartive Transaction. @Transactional marks Transaction Attributes on the spring beans. With Declarative Transactions is a good practise to mark all the methods as Transaction Mandatory, and mark the specific methods with different transaction levels.
EJB would throw an IllegalStateException if one tries to mark transaction for rollback using setRollbackOnly on methods marked Supports, NotSupported or Never. 

## XA Transaction:
XA Transaction was once a solution to a lot of situatitons that demanded distributed transactions. It involves two phase commits - the prepare phase and the commit phase. This is kind of all the resources managers voting for commit before the transaction manager decides it has is issue the transaction commit. However it comes with a lot of issues. The main one being performance. XA Transaction is usually a bottleneck to backend services. This also has a significant effect on the ability of the services to scale. People who have used XA Transactions would have encountered the  Heuristic Rollback Exception or the Heuristic Mixed Exception. This usually happens when the commit phase encounters issue while trying to commit. These exceptions usually require manual intervention or a auto-recovery mechanism to restore consistency. Alternatives to XA transaction can be based on designing the system to be eventually consistent, idempotent and making use of reliable messaging channels. See Eventual Consistency.

## Transaction Design Pattern:
### Alternative to XA Transactions: 
#### Eventual Consistency:
Even for co-ordinating many remote resource XA Transaction is not a bullet proof solution. HeuristicRollBack Exception although rare would require a user to resubmit the request. Even worse HeuristicMixedException would compromise the consistency and might require manual intervention. XA Transaction is not very performant and thus recommended to be avoided wherever possible. An alternate solution is Eventual Consistency. Let's consider a system that whose data has to be consistent over two databases owned by its downstream system. We could introduce a messaging queue (which would send messages which is persistent in nature) between the two systems. A unique transaction id could be used to ensure idempotency i.e. duplicate message (or duplicate delivery os same transaction) would have no side effects on the state of such systems. Thus even if the acknowledgment is not sent back to a Message Broker after updating the database, a redelivered message would not compramise the consistency of data due to the idempotent nature of the system. 
If the consistency is to be maintained across two schemas of same database, one could use one connection (and thus same transaction) instead of having to resort on XA Transactions. This ensures good performance while maintaining consitency.
If the data is to be consistent across two different databases, one could introduce a transaction table to maintain intermediate state in ordered to achieve eventual consistency again by introducing idempotency. 
NoSql Databases don't guarantee ACID properties, instead they are BASE-ic (Basically Atomic Soft state Eventually Consistent).  

### CAP Theorm: 
CAP Theorm states that for any distributed database it is not possible for it be Consistent, Highly Available and Partition tolerant all at the same time. It would have to trade one for the other two. 

--Yet to be completed ---
