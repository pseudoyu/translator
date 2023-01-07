# 为什么我们需要一个数据库连接池？—— 每个程序员的必修课

- 原文地址：<https://medium.com/javarevisited/why-do-we-need-a-database-connection-pool-every-programmer-must-know-9f90e7c8e5af>
- 原文作者：Dineshchandgr
- 本文永久链接：<https://github.com/gocn/translator/blob/master/2023/w01_why_do_we_need_a_database_connection_pool.md>
- 译者：[pseudoyu](https://github.com/pseudoyu)
- 校对：[小超人](https://github.com/focozz)

大家好，通过这篇文章我们将了解数据库连接和它们的生命周期。然后，我们将看看[连接池](https://javarevisited.blogspot.com/2012/06/jdbc-database-connection-pool-in-spring.html)的内部结构，以及为什么我们需要使用它；然后，我们将了解连接池位置的设计模式；接着，我们将探究数据库连接池可能产生的性能问题，并在文章的最后学习 Java 中常用的连接池框架。

![with_connection_pool](../static/images/2023/w01_why_do_we_need_a_database_connection_pool/with_connection_pool.webp)

## 什么是数据库连接

任何软件程序都需要将数据存储在数据库中，为了使应用程序与**数据库服务器**交互，我们需要数据库连接。**数据库连接**只是软件程序与数据库服务交互的一种方式，我们通过数据库连接向数据库发送**命令（SQL）**，并以[结果集]（<http://www.java67.com/2018/01/jdbc-how-to-get-row-and-column-count-from-resultset-java.html>）的形式从数据库获得响应。

![what_is_database](../static/images/2023/w01_why_do_we_need_a_database_connection_pool/what_is_connection_database.webp)

与应用服务器不同，数据库程序通常运行在专用**数据库服务器**中。数据库程序运行在数据库服务器的一个特定端口，应用程序服务器可以通过这个端口发送命令和接收数据。例如：[MySQL 数据库](https://medium.com/javarevisited/top-5-courses-to-learn-mysql-in-2020-4ffada70656f)程序在数据库服务器上运行 3306 默认端口，这与后端应用程序运行在 8080 端口的方式完全相同。

每当像浏览器或手机这样的客户端向后端应用程序请求数据时，后端应用程序需要与数据库交互以获取数据并响应客户端。

如果一个后端应用程序想要连接到数据库服务器程序，它需要通过 [TCP-IP 协议](https://medium.com/javarevisited/5-best-books-and-courses-to-learn-computer-networking-tcp-ip-and-udp-protocols-5a0e4dce75fa)进行调用，并需要提供**数据库服务器的 IP 和端口**信息以及连接的证书。应用服务器连接到数据库服务器以获取数据的过程是通过一个叫做**数据库连接**的机制实现的。

要建立一个数据库连接，我们需要提供数据库 URL（包含主机、端口、数据库名称） 驱动程序、用户名和密码等信息，如下所示

```yaml
db_url      = jdbc:mysql://HOST/DATABASE
db_driver   = com.mysql.jdbc.Driver
db_username = USERNAME
db_password = PASSWORD
```

一旦创建了与数据库的连接，它可以在任何时候被打开和关闭，我们也可以配置超时时间等。

如果没有开放的连接，就不能与[数据库](https://medium.com/javarevisited/8-free-oracle-database-and-sql-courses-for-beginners-f4e9b25b33c4)进行通信。创建数据库连接是一项昂贵的操作，因为涉及很多步骤。我们对数据库连接的操作可能会破坏应用程序，甚至使整个应用程序中断。

## 数据库连接的生命周期

在详细了解了上述数据库连接后，我们来看看连接的**生命周期**，即通过应用服务器创建与数据库连接的各个环节

![connectionlifecycle](../static/images/2023/w01_why_do_we_need_a_database_connection_pool/connectionlifecycle.webp)

1. 通过连接配置字符串与数据库**创建连接**
2. 在建立连接前通过连接配置字符串中的信息对用户身份进行鉴权
3. **创建并打开一个 TCP Socket** 用于读/写数据
4. **通过 Socket 发送/接收数据**
5. 关闭**数据库连接**
6. 关闭**TCP Socket**

与数据库系统建立连接的步骤很多，是一个昂贵而耗时的操作。如果你建立的是一个**实时连接**，**应用程序将挂起**，用户也将遇到页面加载缓慢的情况。此外，如果你的应用程序有大规模的用户，而你为每个请求打开一个连接，同时连接的数量就会增加，这反过来又会增加**CPU 和内存资源消耗**，非常危险。

That is the reason we have **Connection Pools** by which we don't create new connections every time but reuse the existing connections. Without a connection pool, a **new database connection** is created for each client.

Let us look at what happens every time a Database Connection is created

![practicalli-clojure-webapps-database-postgres-no-connection-pool](../static/images/2023/w01_why_do_we_need_a_database_connection_pool/practicalli-clojure-webapps-database-postgres-no-connection-pool.webp)

In the diagram above, we can see that new connections are created to an RDMS database ie. Postgres

When a backend application connects to the PostgreSQL database, the **parent process** in the database server spawns a **worker process** that listens to the newly created connection. Spawning a work process each time also causes additional overhead to the database server. As the number of simultaneous connections increases, the CPU and memory resources of the database server also increase which could crash the database server.

## What is a Database Connection Pool

We saw above that it is inefficient for an application to create, use, and close a database connection whenever it needs to interact with the database. **Connection Pool** is a technique to address the problems associated with creating connections on the fly and to help to improve system performance.

![practicalli-clojure-webapps-database-postgres-connection-pool](../static/images/2023/w01_why_do_we_need_a_database_connection_pool/practicalli-clojure-webapps-database-postgres-connection-pool.webp)

Connection Pool is a **pool of database connections** that can be created ahead of time on application startup and then share the connection (*instead of creating a new one*) among the applications that need to access the database.

When the application is initialized, the provider creates a default connection provided **eg: 10 per server instance** and keeps them in its pool. This DB connection pool resides in the **Application Server’s memory**. When the application needs the connections, these connections from the pool are recycled as creating new connections for every request is a costly operation.

The connection object obtained from the connection pool is a **wrapper around the actual database connection** and the application that uses the connection from the pool is hidden from the underlying complexity. These connections are managed by a **Pool Connection Manager** which is responsible for managing the lifecycle of a connection inside the connection pool.

The approach of Connection Pool encourages opening a connection in an application only when needed and closing it as soon as the work is done without holding the connection open for the entire life of the application. With this approach, a relatively small number of connections can service a large number of requests, which is also known as **Multiplexing**.

The concept of a **Connection Pool** is similar to a **Server Thread Pool** or a [String Pool](http://javarevisited.blogspot.sg/2016/07/difference-in-string-pool-between-java6-java7.html) that facilitates the reusability of already created objects to save the overhead of creating them again thereby resulting in better application performance.

## How is a Database Connection reused from the Connection Pool

The below diagram clearly denoted how the clients use the connections from the pool.

![connection_pool](../static/images/2023/w01_why_do_we_need_a_database_connection_pool/connection_pool.webp)

## Why do we need a Connection Pool

Database connections are pooled for several reasons:

- Database connections are relatively expensive to create, so rather than create them on the fly we opt to create them beforehand and use them whenever we need to access the database.
- The database is a shared resource so it makes sense to create a pool of connections and share them across all business transactions.
- The database connection pool limits the amount of load that you can send to your database.

## Where to place the Database Connection Pool

There are two common ways of placing the Database Connection pool as shown below

### 1. Database Connection pool at the Client level

This is the default approach **in which the Database Connection Pool resides in the memory of a Server / Microservice application**. Whenever the particular server is up, it creates the specified connection and places it in the pool inside its memory. These connections can only be used for the requests that hit this server instance and cannot be used by other microservices. Likewise, every [microservice](https://medium.com/javarevisited/10-best-java-microservices-courses-with-spring-boot-and-spring-cloud-6d04556bdfed) instance has its own connection pool

![where_to_place_database_connection_pool](../static/images/2023/w01_why_do_we_need_a_database_connection_pool/where_to_place_database_connection_pool.webp)

#### Advantages

- Low latency since the pool is on the same box as the requester
- Better security since the connections are constrained to one client

#### Drawbacks

- It can be difficult to monitor and control connections if we use too many microservices

### 2. Shared Database Connection pool as a separate middleware

In this approach, we have a **connection pool in a separate middleware** or in the database server instance to manage the Database connection pool in a centralized manner.

The connections are created in the Connection pool by software like **PgBouncer** and all the microservice instances will share those.

![connection_pool_middlewares](../static/images/2023/w01_why_do_we_need_a_database_connection_pool/connection_pool_middlewares.webp)

#### Pros

- Flexible — database can be swapped out
- Centralized control of connections, which makes it easier to monitor and control connections

#### Cons

- Introducing a new layer. could add latency
- Single point of failure for database calls across all clients
- Potential security issues since you are sharing connections between layers

The choice of where to place the connection pool depends on the specific needs. If your application is small, then go with the 1st approach to place it inside the microservice instances and once the application grows big, you can move the connection pool to a centralized place to manage it easily.

## Performance Issues With Connection Pools

We pool connections to reduce the load on the database because otherwise we might saturate the database with too much load and bring it to a halt. The point is that not only do you want to pool your connections, but you also need to **configure the size** of the pool correctly.

If you **do not have enough connections**, then **business transactions** will be forced to wait for a connection to become available before they can continue processing.

If you have **too many connections**, however, then you might be sending too much load to the database and then all business transactions across all application servers will suffer from slow database performance. The trick is finding the middle ground

The main symptoms of a database connection pool that is sized too small are increased response time across multiple business transactions, with the majority of those business transactions waiting and the symptoms of a database connection pool that is sized too large are increased response time across multiple business transactions, with the majority of those business transactions waiting on the response from queries, and high resource utilization in the database machine.

An application failure occurs when the connection pool overflows. This can occur if all of the connections in the pool are in use when an application requests a connection. For example, the application may use a connection for too long when too many clients attempt to access the website or one or more operations are blocked or simply inefficient.

A connection pool helps to reduce CPU and memory usage but it must be used efficiently. The fixed set of connections is called the **pool size** and it is recommended to test the size of the pool used during **integration tests** to find the **optimal value per application** or per server instance.

## Connection pool implementations for Java

The following are some of the Database Connection pool implementations for Java. When used as a library within the application, these frameworks will take care of the connection pool for the application.

- **Apache Commons DBCP2** — a JDBC Framework based on Commons Pool 2 that provides better performance getting Database Connections through a JDBC Driver, and has JMX Support, among other features.
- **Tomcat JDBC** -  Supports highly concurrent environments and multi-core/CPU systems.
- **pgBouncer** - a lightweight, open-source middleware connection pool for PostgreSQL.
- **HikariCP** - Fast, simple, lightweight, and reliable. This is the default one for the Spring Boot applications using Java. The size of the library is just 130Kb.
- **c3p0** - an easy-to-use library for making traditional JDBC drivers

There are various Connection Pool libraries for other languages also. Moreover, you can also build your own connection pool if you need.

## Summary

In this article, we looked at what is **Database connection and its life cycle**. Then we saw the drawbacks of creating connections on the fly and then saw the need to use a **Database Connection Pool**. We also looked at the design patterns on where to place the connection pool. We have then looked at the **performance issues** that can arise from the Database connection pool and concluded the article by looking at the common connection pool frameworks used in Java.
