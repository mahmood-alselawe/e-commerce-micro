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


# Part 1: Project Setup (config server)

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

3. application.yml : `config Setup`

```yaml
server:
  port: 8888
spring:
  profiles:
    active: native
  application:
    name: config-server
  cloud:
    config:
      server:
        native:
          search-locations: classpath:/configurations

#classpath mean tell to spring boot you should search in resources in directory called configurations
# to read all configurations that belong to another services in you app
#  and also to allow any service when it connect to config server to take His own configuration
# in resources create folder-directory to put all other configuration for another services
```

# to enable config server works add  @EnableConfigServer

```
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(ConfigServerApplication.class, args);
	}

}
// after this run it and see if have any error 
```
# Part 2: Project Setup (eureka server)

1. Spring Initializer : [Spring Initializr](https://start.spring.io)

2. Dependency : `Config Client` , `Eureka Server`

```
	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-config</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>
``` 
3. application.yml : `config Setup`

1. step one:
## go to config server in the directory that you create it create file . yml name should same name of services and put all config that want 
### this path where you should to create your file ==> /main/resources/configurations/discovery-service.yml
```
eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false # you prevnt eureka to register itSelf in server 
    fetch-registry: false # you prevnt eureka to fetch any info from itSelf to take it in  server 
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
server:
  port: 8761

```
2. step two:
 in application.yml : `config Setup`
```
spring:
  config:
    import: optional:configserver:http://localhost:8888 # using url once the app is run will take configuration from config server
  application:
    name: discovery-service # name of file should  in config server should be same think otherwise will get error

```
# to enable eureka server works add  @EnableEurekaServer

```
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer
public class DiscovryApplication {

	public static void main(String[] args) {
		SpringApplication.run(DiscovryApplication.class, args);
	}
	// run this services see if it work or not
	// http://localhost:8761 test if work or not 

}
```
# Part 3: Project Setup (Cutsomer server)

1. Spring Initializer : [Spring Initializr](https://start.spring.io)

2. Dependency : `Config Client` , `Eureka Discovery Client` , `Lombok ` , `Spring Data MongoDB ` ,`Validation ` , `spring web`

```
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-mongodb</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-validation</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-config</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
		</dependency>

		<dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
			<optional>true</optional>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>io.zipkin.reporter2</groupId>
			<artifactId>zipkin-reporter-brave</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
		<dependency>
			<groupId>io.micrometer</groupId>
			<artifactId>micrometer-tracing-bridge-brave</artifactId>
		</dependency>
	</dependencies>
```
3. application.yml : `Customer Setup`

- step one:
## go to config server in the directory that you create it create file . yml name should same name of services and put all config that want 
### this path where you should to create your file ==> /main/resources/configurations/customer-service.yml
```
spring:
  data:
    mongodb:
      username: alibou
      password: alibou
      host: localhost
      port: 27017
      authentication-database: admin
      database: customer
  application:
    name: customer-services
server:
  port: 8090

```

- step two
 in application.yml : `config Setup`
```
spring:
  config:
    import: optional:configserver:http://localhost:8888 # using url once the app is run will take configuration from config server
  application:
    name: customer-services # name of file should  in config server should be same think otherwise will get error

```

## small preif about no-sql and mongoDB

NoSQL is a type of database management system (DBMS) use to handle handle and store large volumes of unstructured and semi-structured data
Unlike traditional relational databases that use tables with pre-defined schemas to store data,
NoSQL databases use flexible data models that can adapt to changes in data structures and are capable of scaling horizontally to handle growing amounts of data.

## SQL (Structured Query Language)

- **Definition**: SQL databases are relational databases that use structured query language for defining and manipulating data.
- **Data Structure**: rows and columns
- **Schema**: predefined schema, fixed
- **Transactions**: Supports ACID (Atomicity(all or no think), Consistency, Isolation , Durability) properties, which ensure reliable transactions.
- **scaling**: vertical (add more power).
- **use case**: structured date  with clear realtions.
- **Examples**: MySQL, PostgreSQL, Oracle, Microsoft SQL Server.

## NoSQL (Not Only SQL)
- **Definition**: NoSQL databases are non-relational databases designed to handle a wide variety of data types and large volumes of data.
- **Data Structure**:  key-value pairs, documents, graphs, or wide-column stores
- **Schema**:  flexible schema
- **Transactions**: Typically do not fully support ACID properties; instead, they may provide eventual consistency, which allows for higher performance and availability.
- **scaling**: horizental scaling (add more server)
- **use case**: un-structured date or sami-structured.
- **Examples**: MongoDB, Cassandra, Redis, Couchbase.

## Summary

- SQL databases are ideal for structured data and complex queries, while NoSQL databases excel in handling unstructured data and providing scalability.
- Choose SQL when data integrity and relationships are crucial, and opt for NoSQL when flexibility and speed are more important.

# Create model for customer-service

```
import lombok.*;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

@AllArgsConstructor
@NoArgsConstructor
@Getter
@Setter
@Builder
@Document
public class Customer {

    @Id
    private String id;
    private String firstName;
    private String lastName;
    private String email;
    private Address address;

}
```
## address class
-  class address and Customer create it in package called model
```
import lombok.*;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

@AllArgsConstructor
@NoArgsConstructor
@Getter
@Setter
@Builder
@Document // this use to tell spring mapping this model to Document
public class Customer {

    @Id
    private String id;
    private String firstName;
    private String lastName;
    private String email;
    private Address address;
//  Mapping to Database document using Spring Data MongoDB not Hibernate
} 
```

# to connect your app wtih mongoDB and get abiltey to interact with it
```
package com.takarub.ecommerce.repository;

import com.takarub.ecommerce.model.Customer;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.data.repository.CrudRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface CustomerRepository extends MongoRepository<Customer, String> {
//  @Id
//    private String id; the type of id shoud same think in extends MongoRepository<Customer, String> 
// here you can defin and methods to retrive and specifi data 
}
```
# create services layer 
-- why we create services layer 
1.  Reusability Business logic often needs to be reused across different parts of the application.
2.  Separation of Concerns  The controller is primarily responsible for handling HTTP requests 

```
package com.takarub.ecommerce.services;

import com.takarub.ecommerce.dto.CustomerRequest;
import com.takarub.ecommerce.dto.CustomerResponse;
import com.takarub.ecommerce.exception.CustomNotFoundException;
import com.takarub.ecommerce.model.Customer;
import com.takarub.ecommerce.repository.CustomerRepository;
import lombok.RequiredArgsConstructor;
import com.takarub.ecommerce.mapper.CustomerMapper;
import org.apache.commons.lang.StringUtils;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class CustomerServices {

    private final CustomerRepository repository;

    private final CustomerMapper mapper;

    public String createCustomer(CustomerRequest request) {
        var customer = repository.save(mapper.toCustomer(request));
        return customer.getId();
    }

    public void updateCustomer(CustomerRequest request) {
       var customer = repository.findById(request.Id())
               .orElseThrow(()-> new CustomNotFoundException(
                       String.format("Customer with id '%s' not found", request.Id())
               ));
       // this methods for checks request  is isNotBlank
       mergerCustomer(customer,request);
       repository.save(customer);

    }

    private void mergerCustomer(Customer customer, CustomerRequest request) {
        if (StringUtils.isNotBlank(customer.getFirstName())) {
            customer.setFirstName(request.firstName());
        }
        if (StringUtils.isNotBlank(customer.getLastName())) {
            customer.setLastName(request.lastName());
        }
        if (StringUtils.isNotBlank(customer.getEmail())) {
            customer.setEmail(request.email());
        }
        if (customer.getAddress() != null) {
            customer.setAddress(request.address());
        }
    }

    public List<CustomerResponse> findAll() {
        return repository.findAll()
                .stream()
                .map(mapper::fromCustomer)
                .collect(Collectors.toList());
    }

    public Boolean exist(String customerId) {

        return repository.findById(customerId)
                .isPresent(); // return true if user is exist else return false

    }

    public CustomerResponse findById(String customerId) {
        return repository.findById(customerId)
                .map(mapper::fromCustomer)
                .orElseThrow(() -> new CustomNotFoundException(customerId));
    }

    public void deleteCustomer(String customerId) {
        repository.deleteById(customerId);
    }
// 



}

```
- here in return type we use CustomerResponse and ues in boy request CustomerRequest this called dto(data transfier object)
why use to  for example by using it we can controller what want to return and what We don't want return same think for boy request
# CustomerResponse DTO
```
package com.takarub.ecommerce.dto;

import com.takarub.ecommerce.model.Address;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotNull;
// i use record instead because of did not to getter and setter etc...
public record CustomerResponse(
        String Id,

        String firstName,

        String lastName,

        String email,

        Address address
) {


}
```
# CustomerRequest DTO

```
package com.takarub.ecommerce.dto;

import com.takarub.ecommerce.model.Address;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotNull;

public record CustomerRequest(
        String Id,
        @NotNull(message = "Customer first name is Required")
        String firstName,
        @NotNull(message = "Customer last name is Required")
        String lastName,
        @NotNull(message = "Customer email is Required")
        @Email(message = "customer email is not valid email address")
        String email,
        Address address
) {
}
```

# Mapper to convert dto to model or model to dto
-- when save model in database you specified what type of model that want save
-- its not allow to you to save CustomerRequest in you database its not allow even if its simmiler in attrubute 
-- so you should to convert it using mapper class 
```
package com.takarub.ecommerce.mapper;

import com.takarub.ecommerce.dto.CustomerRequest;
import com.takarub.ecommerce.dto.CustomerResponse;
import com.takarub.ecommerce.model.Customer;
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;

@Component
public class CustomerMapper {


    public Customer toCustomer(CustomerRequest request) {
        if (request == null) {
            return null;
        }
        return Customer
                .builder()
                .id(request.Id())
                .firstName(request.firstName())
                .lastName(request.lastName())
                .address(request.address())
                .email(request.email())
                .build();
    }

    public CustomerResponse fromCustomer(Customer customer) {
        return new CustomerResponse(
                customer.getId(),
                customer.getFirstName(),
                customer.getLastName(),
                customer.getEmail(),
                customer.getAddress()
        );
    }
}
```

# Handle Exception

### How Spring Boot Handles Exceptions

1. exception happened : When an exception is thrown during the execution of a controller method, Spring starts searching for exception handlers 

2. Global Exception Handling : If no handler for the exception is found within the controller itself, Spring will look for a class annotated with @ControllerAdvice or @RestControllerAdvice.These annotations allow centralized exception handling across the application.

3.Matching the Exception : Inside these advice classes, you define methods annotated with @ExceptionHandler. Spring will match the thrown exception to the corresponding method based on the exc type

4. Response Generation: When a matching method is found, the method processes the exception and returns a response (e.g., a ResponseEntity).


```
package com.takarub.ecommerce.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.HashMap;

@RestControllerAdvice
public class GlobalExceptionHandle {

    @ExceptionHandler(CustomNotFoundException.class)
    public ResponseEntity<String> handleCustomNotFoundException(CustomNotFoundException e) {
        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(e.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleCustomNotFoundException(MethodArgumentNotValidException e) {
        var errors = new HashMap<String, String>();
        e.getBindingResult().getFieldErrors().forEach(error -> {
            var field = ((FieldError)error).getField();
            var errorMessage = error.getDefaultMessage();
            errors.put(field, errorMessage);
        });

        return ResponseEntity
                .status(HttpStatus.BAD_REQUEST)
                .body(new ErrorResponse(errors));

    }
}
``` 

```
package com.takarub.ecommerce.exception;

import lombok.Data;
import lombok.EqualsAndHashCode;

@EqualsAndHashCode(callSuper=true)
@Data
public class CustomNotFoundException extends RuntimeException {

    private final String msg;

}
```

```
package com.takarub.ecommerce.exception;

import java.util.Map;

public record ErrorResponse(
        Map<String,String> errors
) {


}
```



# Controller Layer

1. the controller layer is a crucial part of your application its responseaple It acts as an intermediary between the user interface (the client) and the business logic or service layer.
2. The Controller receives requests from the client, processes them, interacts with the service layer to execute business logic, and then returns the appropriate response
3. Handle incoming HTTP requests: The Controller maps HTTP requests to specific methods using annotations like @GetMapping, @PostMapping, etc.
```
import com.takarub.ecommerce.dto.CustomerRequest;
import com.takarub.ecommerce.dto.CustomerResponse;
import com.takarub.ecommerce.model.Customer;
import com.takarub.ecommerce.services.CustomerServices;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/v1/customer")
@RequiredArgsConstructor
@Slf4j
public class CustomerController {

    private final CustomerServices services;


    // first api create customer

    @PostMapping
    public ResponseEntity<String> createCustomer(@RequestBody @Valid CustomerRequest request) {
        return ResponseEntity.ok(services.createCustomer(request));
    }

    // to update the customer
    @PutMapping
    public ResponseEntity<Void> updateCustomer(
            @RequestBody @Valid CustomerRequest request) {
        services.updateCustomer(request);
        return ResponseEntity.accepted().build();

    }
    @GetMapping
    public ResponseEntity<List<CustomerResponse>> getAllCustomers() {
        return ResponseEntity.ok(services.findAll());
    }


    // this methods the most important 
    @GetMapping("/exist/{customer-id}")
    public ResponseEntity<Boolean  > existById(@PathVariable("customer-id") String customerId) {
        return  ResponseEntity.ok(services.exist(customerId));
    }

    @GetMapping("/{customer-id}")
    public ResponseEntity<CustomerResponse> getCustomerById(@PathVariable("customer-id") String customerId) {

        return  ResponseEntity.ok(services.findById(customerId));

    }
    @DeleteMapping("/{customer-id}")
    public ResponseEntity<Void> deleteCustomer(@PathVariable("customer-id") String customerId) {
        services.deleteCustomer(customerId);
        return ResponseEntity.accepted().build();
    }


}
```

# Part 3: Project Setup (Product server)


1. Spring Initializer : [Spring Initializr](https://start.spring.io)

2. Dependency : `Config Client` , `Eureka Discovery Client` , `Lombok ` , `Spring Data jpa ` ,`Validation ` , `spring web` , `Flyway Migration`.`PostgreSQL Driver`

   
1.**Flyway**  is a tool that automatically updates and organizes changes to your database, ensuring it evolves along with your application by using simple versioned scripts.
   
**Why We Use Flyway**
1. To version control the database schema. Each time you update the schema (like adding or modifying tables), you can track those changes in migration files.
2. It provides an automated process to ensure the database is always in sync with your application across environments (development, staging, production).
3. It makes sure the database is always consistent and avoids problems from different versions being used.

- **Flyway** is responsible for database schema management and ensures that the tables and schema are created or modified based on migration scripts.
- **Spring Data JPA** is responsible for object-relational mapping (ORM) and converting Java objects into database entities (and vice versa).

# to use Flyway Migration

1. in resources directory create directory called db and inside them another directory called migration otherwise your app with not run correct 

the Path should be like this **resources/db/migration** 


2. V<version_number>__migrationScriptName.sql        ===>    Ex : V1__create_user_table.sql

- V1 is the version number, indicating this script should be executed first in orde.

- Ensure that after the version number comes two underscores (__) and the name is delimited by single underscores.

- In this article, we’ll look at how we can apply the flyway database migration scripts in our Spring Boot Application.

3. V1__init_database.sql create this file with extention sql

```
create table if not exists category
(
    id int not null primary key,
    description varchar(255),
    name varchar(255)


);

create table if not exists product
(
    id int not null primary key,
    description varchar(255),
    name varchar(255),
    available_quantity double precision not null ,
    price numeric(38,2),
    category_id integer
            constraint fk1moodconstraint references category

);

create sequence if not exists category_seq increment by 50;
create sequence if not exists product_seq increment by 50;

```
in case you did not have flay way who is responseaple to create table

1. when set spring.jpa.hibernate.ddl-auto=create-drop in your yml file or properties 
2. Spring Boot configures Hibernate to automatically manage the database schema based on your JPA entity classes when the application starts up
3. Spring Boot scans for classes annotated with @Entity within the specified packages
4. Hibernate, as the JPA provider, reads the entity classes discovered during the scanning process.
5. It uses reflection to access the class annotations (like @Entity, @Table, @Id, etc.) and  determine how to map the class to a database table.

- For example, it looks at the @Table annotation to find out what table name to use, and the @Id annotation to identify the primary key.


**and to use** 

  






