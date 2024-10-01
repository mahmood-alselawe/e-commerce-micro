# Spring Boot Microservices with Eureka, Keycloak, and Gateway i

introduction This project demonstrates a microservices architecture using Spring Boot.

# What is Microservices Architecture?

Microservices is an architectural style where a large application is built as a collection of small, loosely coupled, and independently deployable services. Each service is designed to perform a specific business function, and these services communicate over a network.

# Key Characteristics of Microservices:

- Single Responsibility: Each service focuses on a single function or capability, making it easier to develop, test, and maintain.
- Independent Deployment: Each service is developed independently, allowing teams to work autonomously.
- Decentralized Data Management: Each service can have its own database (SQL or NoSQL), ensuring flexibility in data storage.
- Technology Diversity: Since services are independent, you can use multiple programming languages within the same project (e.g., Java, C#, PHP).
- Communication: Microservices communicate through APIs (typically RESTful) or messaging brokers like Kafka.

# Core Components in Spring Boot Microservices:

1. Config Server
- Centralized configuration management: All configurations for services are stored centrally,
so every service connects to the Config Server to retrieve its configuration.
- spring Cloud Config handles this. For instance, in a large-scale system, you may have multiple instances of the same service running on different ports. Without a centralized config, each instance would require manual changes making the system harder to manage.

2. Eureka Server
- Service Discovery: Eureka allows all your microservices to register themselves so they can discover and communicate with each other dynamically, without hardcoding URLs. The API Gateway also uses Eureka to route requests to the appropriate service.
3. API Gateway
- Single Entry Point: The API Gateway acts as the single entry point for external requests to your microservices system. Rather than exposing each microservice directly, the Gateway routes requests based on URL patterns.
- ecurity with Keycloak: The Gateway integrates with Keycloak for authentication and authorization. It ensures that users can only access services they are permitted to. By connecting with Eureka, the Gateway dynamically discovers microservices, making the system flexible and scalable.

![Microservices Architecture](https://github.com/mahmood-alselawe/e-commerce-micro/blob/master/Screen%20Shot%202024-09-29%20at%2010.21.38%20PM.png)

Project Structure
This project contains the following services:
1. Config Server : Centralized configuration management.
2. Eureka Server : Service discovery server.
3. API Gateway : Handles routing and integrates with Keycloak for security.
4. Customer Service : Connects to Eureka, retrieves configuration from the Config Server, and uses MongoDB for data.
5. Product Service : Connects to Eureka, retrieves configuration from the Config Server, and uses PostgreSQL for data.
6. Order Service : Connects to Eureka, retrieves configuration from the Config Server, and uses PostgreSQL. Communicates with Product Service via RestTemplate to check product 	 
   stock and uses OpenFeign to check customer details.
7. Payment Service : Handles payments for orders, communicates with Order Service, and sends Kafka messages to the Notification Service.
8. Notification Service : Listens to Kafka for order and payment confirmations and sends notifications to the customer (via email/SMS).
9. Zipkin : Used for distributed tracing to monitor and visualize request paths across microservices.
10. Keycloak : Secures access to the microservices. Both direct service access and Gateway access are secured using Keycloak.


# now let deep dive into this project 

let statrt with **docker-compose.yml** file

# What is docker-compose.yml file 

- Docker is an open-source platform that enables developers to automate the deployment, scaling, and management of applications in lightweight 

Docker Compose is a tool for defining and running multi-container applications 
It allows you to configure all the services, networks, and volumes that are part of the application in a single file. 
Docker Compose makes it easier to manage applications that consist of multiple containers by defining everything in a declarative YAML format.


`services:
   testtest`




