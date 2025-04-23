# Implementing an annotation-based <b>@Transactional</b>-like mechanism for DynamoDB transactions
## Chapter One — Some theory

Most engineers who have worked on projects utilizing the Spring Framework have also employed <a href="https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative.html">Spring’s Declarative Transaction Management mechanism</a> to ensure the atomicity of their database-related operations. The AOP-based approach is lightweight and elegant, with minimal impact on the codebase.
It’s possible that, because declarative transactions are so non-intrusive to our code, we are often not motivated enough to look under the hood and understand the implementation details of this widely used Spring feature.

In this short series of articles, I will explore the ways <b>@Transactional</b> operates, the aspects (literally!) on which it is based, and how we could create a similar declarative mechanism for the database that functions differently from databases supported by Spring’s <b>@Transactional</b>.

## Quick summary of how the transaction mechanism (in Spring) works
Suppose you have ever used a CLI client to interact with your database. In that case, you might have been using a transaction, just in case something went wrong, so that you could perform a rollback (especially on production).

It might have looked more or less like this:
```
>> OPEN the connection to the database...
>> BEGIN;
>> DELETE FROM <table> WHERE id = `id`
>> COMMIT; // or ROLLBACK;
>> CLOSE the connection.
```

## Does the Spring Transactional mechanism also send out BEGIN and COMMIT when you use the <b>@Transactional</b> annotation?
<b>No… and somehow yes</b>, it depends on the implementation. It does not directly call BEGIN, COMMIT, or ROLLBACK. It uses APIs like JDBC and its methods instead, which ultimately translate to database-specific statements depending on the database driver used.
All transaction-related calls in Spring Framework go through the selected implementation of the <a href="https://github.com/spring-projects/spring-framework/blob/main/spring-tx/src/main/java/org/springframework/transaction/PlatformTransactionManager.java">(Platform)TransactionManager</a> interface, which, in a simplification, will execute the JDBC (JPA, Hibernate, etc.) equivalent of BEGIN, COMMIT, or ROLLBACK, among other things like setting isolation level, transaction propagation mode, etc.
For more information on how Spring Transaction Management works, I recommend reading <a href="https://www.marcobehler.com/guides/spring-transaction-management-transactional-in-depth">„Spring Transaction Management: @Transactional In-Depth” by Marco Behler.</a>

## Does Spring’s transaction manager keep the database connection open?
<b>Yes.</b> The transaction manager obtains the database connection from the connection pool and uses this connection to interact with a database.
The connection in Spring is usually obtained through a DataSource object provided to the transaction manager, which isconfigured (often automatically) for your database of choice.

> Notes about DynamoDB: communication with DynamoDB using AWS SDK for Java is designed as external API calls, so there will be no open, direct connection to the database. There will also be no BEGIN, COMMIT, or ROLLBACK commands known from the examples above. Write and read transactions were designed as API calls that accept arguments as a request body. The request body defines the scope of the transaction. There is also no ROLLBACK as the transaction request either succeeds or fails. ACID guarantees are kept on the AWS infrastructure side under a set of conditions listed here.

## Main functionalities of the declarative transaction mechanism in the Spring Framework
Adding the <b>@Transactional</b> annotation on the class or method level ensures that all database operations executed will be part of a thread-bound transaction.
The declarative transaction mechanism wired by the <b>@Transactional</b> annotation has many advantages and functionalities beyond being a „simple” transaction wrapper.
1. eliminates the need for programmatic transaction handling,
2. provides a mechanism for several transaction propagation types: REQUIRED, REQUIRES_NEW, MANDATORY, etc.
3. rollback management,
4. timeout configuration,
5. read-only operations and performance optimizations,
6. … and then some.

> Notes about DynamoDB: I will show you how to implement most of those points using a mechanism similar to the one known from Spring. I will focus on recreating a <b>@Transactional</b>-like annotation and AOP interceptor, and follow up with example implementations of most of the propagation mechanisms and read-only transactional calls to DynamoDB using the API provided by the AWS SDK.

## How is the <b>@Transactional</b> annotation-based mechanism implemented?
The solution is based on the AOP (Aspect Oriented Programming) proxies. Methods or classes that match a defined pointcut (in that case, methods or classes annotated with the <b>@Transactional</b> annotation) are being picked up and intercepted, allowing programmers to define additional actions that may happen before, after, or around the invoked method.

## We had some theory, now it is time for some code…
In the next article, we will start implementing our solution for DynamoDB.  First, we will create a <b>@Transactional</b>-like annotation, then an Aspect responsible for beginning, propagating (starting with PROPAGATION_REQUIRED), and committing a DynamoDB transaction. Additionally, we will explore some new features coming to the Java ecosystem with ScopedValues.

Stay tuned for: <b>Chapter Two — The basic implementation!</b>

