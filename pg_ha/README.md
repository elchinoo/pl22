# PostgreSQL High Availability With Patroni

This tutorial is intended to show how to build a High Availability (HA) setup using PostgreSQL Streaming Replication and Patroni.
 
We start from the premise that every architecture and deployment depends on the customer requirements, application demands for high availability, the estimated level of usage, and the physical infrastructure.  For example, one may have an application that is read-intensive, write-intensive, or a mix of both, which demands 99% or 99.999% availability SLA, still, the nodes are physically isolated sitting on different continents. We will see how to implement some of those requirements and also why some of the combinations are very hard to be fulfilled.
 
We also acknowledge that there are multiple tools and architectures available for managing the high availability of a PostgreSQL cluster. Each of them has its own pros and cons, but this tutorial will focus on Patroni used with streaming replication as the replication method. We'll present a simple but effective architecture, explain the reasons why we are using this architecture, and then deploy it.

I would like to emphasize that understand the reasons why we chose the architecture, the problems it solves, and the problems it doesn't solve and how to improve it is much more important than how to build it. There are tons of tutorials and posts on internet showing how to build the architecture we'll discuss here, but only a few of them discuss the reasons.

## Architecture

As we can see below we'll implement a cluster with 3 nodes, 1 primay and 2 replicas: 

![PostgreSQL HA architecture with Patroni and HAProxy](pg_ha_patroni.png.png)

