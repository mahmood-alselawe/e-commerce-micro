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


# Docker Compose Setup

```yaml
version: '3.9'
# Defines the containers that will be part of your application.
# Each container is defined as a "service."
services:   
  postgresql:        # name of service 
    container_name: ms_pg_sql  # The name of the PostgreSQL container (ms_pg_sql).
    image: postgres # Uses the official postgres image from Docker Hub.
    environment: # here we can define any attribute that i needed it in my container
      POSTGRES_USER: mood
      POSTGRES_PASSWORD: mood
      PGDATA: /data/postgres # here where we want to save data 
    volumes: # the docker  containers are stateless by design, meaning if a container stops or is deleted,	   
      - postgres:/data/postgres # all the data inside it will be lost. Volumes provide a way to persist this data
    ports:
      - "5432:5432"  # the port for services - "external_port:internal_port"
    networks: #Connects to the custom microservices-net network.
      - microservices-net
    restart: unless-stopped #The container will restart if it fails, except if it was stopped manually (unless-stopped).

  pgadmin:
    container_name: ms_pgadmin
    image: dpage/pgadmin4
    environment:
      PGADMIN_DEFAULT_EMAIL: ${PGADMIN_DEFAULT_EMAIL:-pgadmin4@pgadmin.org}
      PGADMIN_DEFAULT_PASSWORD: ${PGADMIN_DEFAULT_PASSWORD:-admin}
      PGADMIN_CONFIG_SERVER_MODE: 'False'
    volumes:
      - pgadmin:/var/lib/pgadmin
    ports:
      - "5050:80"
    networks:
      - microservices-net
    restart: unless-stopped

  zipkin:
    container_name: zipkin
    image: openzipkin/zipkin
    ports:
      - "9411:9411"
    networks:
      - microservices-net

  mongodb:
    image: mongo
    container_name: mongo_db
    ports:
      - 27017:27017
    volumes:
      - mongo:/data
    environment:
      - MONGO_INITDB_ROOT_USERNAME=modMon
      - MONGO_INITDB_ROOT_PASSWORD=modMon

  mongo-express:
    image: mongo-express
    container_name: mongo_express
    restart: always
    ports:
      - 8081:8081
    environment:
      - ME_CONFIG_MONGODB_ADMINUSERNAME=modMon
      - ME_CONFIG_MONGODB_ADMINPASSWORD=modMon
      - ME_CONFIG_MONGODB_SERVER=mongodb

  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    container_name: zookeeper
    environment:
      ZOOKEEPER_SERVER_ID: 1
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "22181:2181"
    networks:
      - microservices-net

  kafka:
    image: confluentinc/cp-kafka:latest
    container_name: ms_kafka
    ports:
      - "9092:9092"
    depends_on:
      - zookeeper
    environment:
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
    networks:
      - microservices-net

  mail-dev:
    container_name: ms-mail-dev
    image: maildev/maildev
    ports:
      - 1080:1080
      - 1025:1025

  mysql-kc:
    image: mysql:8.0.27
    container_name: mysql_kc
    ports:
      - "3377:3306"
    restart: unless-stopped
    environment:
      MYSQL_USER: bisky
      MYSQL_PASSWORD: password
      MYSQL_DATABASE: keycloak
      MYSQL_ROOT_PASSWORD: password
    volumes:
      - keycloak-and-mysql-volume:/var/lib/mysql
    networks:
      - microservices-net

  keycloak-w:
    image: quay.io/keycloak/keycloak:25.0.0
    container_name: keycloak_w
    command: start-dev
    ports:
      - "9082:8080"
    restart: unless-stopped
    environment:
      KEYCLOAK_ADMIN: admin1
      KEYCLOAK_ADMIN_PASSWORD: admin@12345
      KC_DB: mysql
      KC_DB_USERNAME: root
      KC_DB_PASSWORD: password
      KC_DB_URL: jdbc:mysql://mysql-kc:3306/keycloak
      KC_FEATURES: token-exchange,admin-fine-grained-authz
      KC_HOSTNAME: localhost
    networks:
      - microservices-net

networks:
  microservices-net:
    driver: bridge

volumes:
  postgres:
  pgadmin:
  mongo:
  keycloak-and-mysql-volume:
```
after we collect all services that i need it to use in my application you sould to test it
for example: 

- keycloak `http://localhost:9082`
- postgresql `http://localhost:5050`


Part 1: Project Setup (config server)

1. Spring Initializer : [Spring Initializr](https://start.spring.io)

2. Dependency : `Config Server` 

```
<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-config-server</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

```
