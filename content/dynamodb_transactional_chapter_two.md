# Implementing an annotation-based @Transactional-like mechanism for DynamoDB transactions
## Chapter Two - the annotation, the aspect, and a bit about write transactions


Motivated by releasing <a href="https://krzysztof-owczarek.github.io/kowczarek.github.io/content/dynamodb_transactional_chapter_one_theory.html">Chapter One - Some theory</a> (short, but important for understanding what we will be doing right now), I follow up quickly with some code that will bring us closer to our goal - an annotation-based declarative transactional mechanism known from the Spring Framework, currently without support for DynamoDB.


Let's unlock the support ourselves! The code will be written in Kotlin with some Java flavours, like ScopedValue. I know that the **ScopedValue** part could be dropped for a full Kotlin solution, but I liked the JEP and wanted to introduce it to you too.

## The @Annotation…

We will start easy with writing an annotation for marking methods, where communication with DynamoDB should be part of a write transaction.


> Note about DynamoDB: Read/Write transactions known from such databases as PostgreSQL are unavailable in DynamoDB (at least not on 24.05.2025). Each direction of transactional data flow has its own AWS SDK API call.


```kotlin
annotation class DynamoWriteTransaction(
    val propagation: DynamoTransactionPropagation = DynamoTransactionPropagation.REQUIRED
)
```


As you may see, I have already included a propagation type for our transaction, with a default value - REQUIRED, as it is also a default in the existing Spring solution. In this chapter, we will cover only this propagation type. The other types will be covered in the next chapters.
Usually, the propagation logic is  handled by a database driver, we do not have any, so we need to write the proper logic ourselves.

## The Transaction…


There is no actual **transaction** in DynamoDB representing a mechanism known from other databases like PostgreSQL (more about how all this works in <a href="https://krzysztof-owczarek.github.io/kowczarek.github.io/content/dynamodb_transactional_chapter_one_theory.html">Chapter One - Some theory</a>). Instead, we have an AWS SDK transactional request that accepts a list of objects to include in a single transaction.


To have a representation of a transaction, I introduced a `DynamoWriteTransactionManager`, an object with the responsibility of wrapping **a single (sic!)** transaction's request builder and exposing an API with basic methods (save, delete, and commit).
The `DynamoWriteTransactionManager` will live as long as the **transaction** it wraps.


```kotlin
@Component
@Scope(value = ConfigurableBeanFactory.SCOPE_PROTOTYPE)
class DynamoTransactionManager(
    transactionRequestBuilderFactory: TransactionRequestBuilderFactory,
    private val enhancedClient: DynamoDbEnhancedClient
) {

    val transactionId: String = UUID.randomUUID().toString()

    private val transactionRequestBuilder = transactionRequestBuilderFactory.builder()

    private val transactionClosed = AtomicBoolean(false)

    /**
     * Transactional request cannot be empty.
     * Boolean will be switched to true after the first element is added to the request.
     */
    private val committable = AtomicBoolean(false)

    fun <T> save(table: DynamoDbTable<T>, entity: T) {
        if (transactionClosed.get()) {
            throw RuntimeException("[Transaction#$transactionId] **transaction** has been already closed.")
        }

        transactionRequestBuilder.addPutItem(
            table,
            entity
        )

        committable.set(true)
    }

    fun <T> delete(table: DynamoDbTable<T>, entity: T) {
        if (transactionClosed.get()) {
            throw RuntimeException("[Transaction#$transactionId] **transaction** has been already closed.")
        }

        transactionRequestBuilder.addDeleteItem(
            table,
            entity
        )

        committable.set(true)
    }

    fun commit() {
        if (transactionClosed.compareAndSet(false, true)) {
            if (committable.get()) {
                enhancedClient.transactWriteItems(transactionRequestBuilder.build())
            } else {
                throw RuntimeException("[Transaction#$transactionId] **transaction** is not committable" +
                        " because transactional request is empty.")
            }
        } else {
            throw RuntimeException("[Transaction#$transactionId] **transaction** has been already closed.")
        }
    }
}
```


The code is not complicated, but I want to describe a few things in greater detail.


The first thing that should catch your eye is registering the `DynamoWriteTransactionManager` bean as prototype-scoped. 


```kotlin
@Scope(value = ConfigurableBeanFactory.SCOPE_PROTOTYPE)
```


The scope has been chosen to fulfill our requirement of an object that should live only as long as the **transaction** it wraps. Because we inject DynamoDbEnhancedClient and TransactionRequestBuilderFactory beans via a constructor, we must register `DynamoWriteTransactionManager` in the Spring Context.

Using the prototype bean for our purpose will be tricky and requires some Spring knowledge, but I will cover it in the last paragraph.

- 
The next thing worth clarifying is the two `AtomicBoolean` variables.
1. `transactionClosed` prevents calling the manager's API after the **transaction** has already been committed, but a reference to the **manager** is still available from the code.
2. `committable` prevents sending out an empty transactional request, which would result in an error response from the DynamoDB API.


## The Aspect…


To wrap our code in a transactional request to DynamoDB, we need to perform some action before the code runs - use an ongoing **transaction** available in our scope or start a new one -  and also act after the code has been executed (commit the transaction).


To achieve this, we need to create an aspect. AOP in Spring can be achieved in at least two ways - Spring AOP and AspectJ. To know more about those two options, please refer to the documentation - <a href="https://docs.spring.io/spring-framework/reference/core/aop.html">Aspect Oriented Programming with Spring</a>.


```kotlin
@Aspect
@Component
class DynamoTransactionAspect(
    private val dynamoTransactionManagerPrototypeFactory: DynamoTransactionManagerPrototypeFactory
) {

    private val log = KotlinLogging.logger {}

    private val dynamoTransactionManagerScopedRef = DynamoTransactionManagerScopedRef.scopedValue

    @Around("@annotation(writeTransaction")
    fun aroundWriteTransaction(
        joinPoint: ProceedingJoinPoint,
        dynamoWriteTransaction: DynamoWriteTransaction
    ): Any? {
        return when(dynamoWriteTransaction.propagation) {
            DynamoTransactionPropagation.REQUIRED -> {
                if (dynamoTransactionManagerScopedRef.isBound) {
                    log.debug { "[Transaction#${dynamoTransactionManagerScopedRef.get().transactionId}] " +
                            "Transaction already in progress..." }
                    return joinPoint.proceed()
                }

                startTransaction(dynamoTransactionManagerScopedRef, joinPoint)
            }
        }
    }

    private fun startTransaction(
        dynamoTransactionManagerScopedRef: ScopedValue<DynamoTransactionManager>,
        joinPoint: ProceedingJoinPoint
    ) = ScopedValue.where(dynamoTransactionManagerScopedRef, dynamoTransactionManagerPrototypeFactory.create()).call<Any, Throwable> {
        log.debug { "[Transaction#${dynamoTransactionManagerScopedRef.get().transactionId}] Starting transaction." }
        return@call joinPoint.proceed().also {
            dynamoTransactionManagerScopedRef.get().commit()
        }
    }
}
```


The DynamoTransactionAspect reads the propagation property from the DynamoWriteTransaction annotation and steers the logic of handling the propagation type according to its requirements. In this chapter, we focus on the default propagation type - REQUIRED.


The propagation type REQUIRED is simple. The code requires that the **transaction** exists, so you use the existing one or create a new one, run the code, and commit the transaction.


- 
 
The **ScopedValue** part is the most interesting one (at least for me!). To understand how the code works, you should first understand how **ScopedValue** works. 

I recommend reading the <a href="https://openjdk.org/jeps/446">JEP-446 ScopedValues (Preview)</a> and <a href="https://docs.oracle.com/en/java/javase/22/docs/api/java.base/java/lang/ScopedValue.html">Javadocs</a> - both are well written and provide all the details you need.


```kotlin
object DynamoTransactionManagerScopedRef {
    val scopedValue: ScopedValue<DynamoTransactionManager> = ScopedValue.newInstance()
}
```


`DynamoTransactionManagerScopedRef` is nothing more than an object wrapper for a variable of type `ScopedValue<DynamoTransactionalManager>`, which is itself a wrapper for a DynamoTransactionalManager object reference.


Why do we need such a wrapper inception?


```kotlin
ScopedValue.where(dynamoTransactionManagerScopedRef, dynamoTransactionManagerPrototypeFactory.create()).call {}
```


This line is crucial. It binds an instance of DynamoTransactionalManager prototype-scoped bean (every time we ask for the bean, we get a new one*) to a named **ScopedValue** that is thread-bound, making the object it wraps accessible from anywhere in the code (as long as it runs on the same thread) without the dangers and pitfalls of ThreadLocal. 

---

> Additionally, it opens up the possibility of using StructuredConcurrency for Java - Kotlin has its own solution for structured concurrency, but that one everybody knows already.
That would not only allow us to make our code more robust, but this solution could also gain an advantage over <a href="https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-decl-explained.html">Spring's @Transactional mechanism</a>, enabling sharing a **transaction** across multiple threads, even for non-reactive code, thanks to <a href="https://docs.oracle.com/en/java/javase/22/docs/api/java.base/java/lang/ScopedValue.html#inheritance">**ScopedValue** inheritance</a> - it has not been tested, it is just an assumption*.

---

## *How to create a prototype-scoped **manager** without Spring DI?

As mentioned before, we would like to access some of the objects registered as Spring beans from our DynamoTransactionalManager. To inject those dependencies into our manager's constructor, we have to register it in the Spring IoC container.
Additionally, we want our **manager** to be a prototype bean, because every time we create it, it will be the wrapper for a completely new DynamoDB transaction.


```kotlin
@Component
class DynamoTransactionManagerPrototypeFactory(
    private val factory: ObjectFactory<DynamoTransactionManager>
) {
    fun create() = factory.`object`
}
```


This factory class does the trick. It is registered as a singleton bean and takes the `ObjectFactory<DynamoTransactionManager>` via constructor injection. That lets us programmatically create prototype beans with all required dependencies wired up by Spring DI. Neat, right?

## Chapter Three - code repository and tests.

I have planned to cover everything in this chapter, but it is already very long.

In the next chapter, I will share a repository with code for the described solution. It will also involve the test suite that will (hopefully) prove that the solution is working as planned.


We will try to cover the next propagation types soon - next step REQUIRES_NEW and **nested transactions!**


Happy coding! 💪
