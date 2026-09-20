# Hotel Reservation Application

## Overview

This project is a full-stack hotel reservation application built with **Java, Spring Boot, Spring Data JPA, H2, and Angular**.

The application provides a REST API for retrieving hotel rooms, determining which rooms are available for a requested date range, retrieving individual rooms, and creating reservations.

The backend follows a layered architecture that separates REST controllers, services, repositories, entities, request/response models, and conversion logic.

## Technologies Used

* **Java 17**
* **Spring Boot 2.7.2**
* **Spring Web**
* **Spring Data JPA**
* **Hibernate**
* **H2 Database**
* **MySQL Connector**
* **Angular 14**
* **Maven**
* **JUnit 5**
* **Docker**

## Project Architecture

The backend is organized into several layers:

```text
Angular Frontend
       │
       ▼
REST Controller
       │
       ▼
Service Layer
       │
       ▼
Repository Layer
       │
       ▼
H2 Database
```

The application also uses converter classes to transform database entities into API response objects and incoming request objects into database entities.

---

## REST API

The main REST controller is:

```text
ReservationResource
```

The base API path is:

```text
/room/reservation/v1
```

This constant is defined in `ResourceConstants`.

### Get Available Rooms

```http
GET /room/reservation/v1?checkin=YYYY-MM-DD&checkout=YYYY-MM-DD
```

The endpoint accepts a check-in and check-out date.

The controller retrieves:

1. All rooms
2. All existing reservations
3. The requested reservation dates

It then compares existing reservations against the requested dates and removes rooms that are already reserved during the requested period.

The remaining rooms are converted into `ReservableRoomResponse` objects and returned to the Angular application.

### Get a Room

```http
GET /room/reservation/v1/{roomId}
```

This endpoint retrieves a specific room by its database ID.

The `RoomRepository` uses Spring Data's `findById()` method to locate the room.

### Create a Reservation

```http
POST /room/reservation/v1
```

The endpoint accepts a JSON `ReservationRequest`.

The request contains:

* Reservation ID
* Room ID
* Check-in date
* Check-out date

The request is converted into a `ReservationEntity`, associated with the selected room, and saved through the `ReservationRepository`.

---

## Entity Layer

### RoomEntity

`RoomEntity` represents a hotel room in the database.

Important properties include:

```java
private Long id;
private Integer roomNumber;
private String price;
private List<ReservationEntity> reservationEntityList;
```

The entity is mapped to the `Room` database table using JPA:

```java
@Entity
@Table(name = "Room")
```

Rooms can have associated reservations through a one-to-many relationship.

---

### ReservationEntity

`ReservationEntity` represents a reservation.

It contains:

```java
private Long id;
private LocalDate checkin;
private LocalDate checkout;
private RoomEntity roomEntity;
```

The reservation is connected to a room through:

```java
@ManyToOne
@JoinColumn(name = "room_id", nullable = false)
```

This allows each reservation to reference the room being reserved.

---

## Repository Layer

The application uses Spring Data repositories to communicate with the database.

### RoomRepository

```java
public interface RoomRepository extends CrudRepository<RoomEntity, Long>
```

This provides standard database operations for rooms without requiring SQL to be written manually.

For example:

```java
roomRepository.findById(id);
roomRepository.findAll();
```

### ReservationRepository

```java
public interface ReservationRepository extends CrudRepository<ReservationEntity, Long>
```

This repository provides database operations for reservations.

Spring Data automatically supplies the implementation at runtime.

---

## Service Layer

The service layer sits between the REST controller and repositories.

### RoomService

`RoomService` defines operations for working with rooms:

```java
RoomEntity findById(long theId);
List<RoomEntity> findAll();
```

`RoomServiceImpl` implements these operations using `RoomRepository`.

The service checks whether a room exists before returning it.

### ReservationService

`ReservationService` provides operations for reservations:

```java
ReservationEntity findLast();
List<ReservationEntity> findAll();
```

`ReservationServiceImpl` uses `ReservationRepository` to retrieve reservation data.

The service abstraction keeps database operations separated from the REST controller.

---

## Converter Layer

The application uses Spring's `Converter` interface to transform objects between different layers.

### ReservationRequestToReservationEntityConverter

This converter transforms an incoming API request into a database entity.

The process is:

```text
ReservationRequest
        │
        ▼
Find Room
        │
        ▼
ReservationEntity
        │
        ▼
Database
```

The converter takes the requested room ID and retrieves the corresponding `RoomEntity`.

It then creates a `ReservationEntity` containing:

* Check-in date
* Check-out date
* Associated room
* Reservation ID when provided

---

### RoomEntityToReservableRoomResponseConverter

This converter transforms a `RoomEntity` into a `ReservableRoomResponse`.

The response contains:

* Room ID
* Room number
* Price
* Self-link

The price is converted from the entity's `String` representation into an `Integer`.

The converter also creates a REST-style self reference:

```text
/room/reservation/v1/{roomId}
```

This gives the frontend a direct reference to the room resource.

---

## Request and Response Models

The project separates API objects from database entities.

### ReservationRequest

Used when the client creates a reservation.

```java
private Long id;
private Long roomId;
private LocalDate checkin;
private LocalDate checkout;
```

### ReservationResponse

Used when returning reservation information.

```java
private Long id;
private LocalDate checkin;
private LocalDate checkout;
```

### ReservableRoomResponse

Used when returning rooms that can be displayed as available.

```java
private Long id;
private Integer roomNumber;
private Integer price;
private Links links;
```

This separation prevents the REST API from needing to expose database entities directly for every operation.

---

## Availability Logic

One of the main pieces of application logic is determining which rooms are available.

The controller first retrieves every room:

```java
List<RoomEntity> allRooms = roomService.findAll();
```

It then retrieves all existing reservations:

```java
List<ReservationEntity> allReservations = reservationService.findAll();
```

Each reservation is compared against the requested check-in and check-out dates.

Rooms that conflict with an existing reservation are removed from the available-room list.

The remaining rooms are returned to the frontend.

Conceptually:

```text
All Rooms
    │
    ▼
Compare Against Reservations
    │
    ├── Conflict → Remove Room
    │
    └── No Conflict → Keep Room
                         │
                         ▼
                 Available Rooms
```

---

## Database Configuration

The application is configured to use an H2 database.

The database configuration is located in:

```text
src/main/resources/application.properties
```

The project uses a file-based H2 database:

```properties
spring.datasource.url=jdbc:h2:file:~/spring-boot-h2-d387F
```

Hibernate is configured to update the database schema automatically:

```properties
spring.jpa.hibernate.ddl-auto=update
```

SQL generated by Hibernate can also be displayed:

```properties
spring.jpa.show-sql=true
```

The H2 console is enabled at:

```text
/h2-console
```

---

## Angular Frontend

The project includes an Angular frontend located in:

```text
src/main/UI
```

The Angular application was generated using Angular CLI 14.1.3.

The frontend communicates with the Spring Boot backend through HTTP requests.

The compiled Angular application is placed into the Spring Boot application's static resources so that the backend can serve the frontend.

The production build is represented by files such as:

```text
src/main/resources/static/index.html
src/main/resources/static/main.*.js
src/main/resources/static/polyfills.*.js
src/main/resources/static/runtime.*.js
```

---

## Maven Build

The project uses Maven for dependency management and building.

The primary configuration is located in:

```text
pom.xml
```

The project uses Java 17 and includes dependencies for:

* Spring Boot
* Spring Web
* Spring Data JPA
* H2
* MySQL
* Validation
* Testing
* Lombok

Maven is also configured to build the Angular frontend during the Maven lifecycle.

---

## Testing

The project includes a Spring Boot test:

```java
@SpringBootTest
class D387SampleCodeApplicationTests {

    @Test
    void contextLoads() {
    }

}
```

This test verifies that the Spring application context can successfully load.

---

## Docker and Cloud Deployment

The application is designed to be packaged as a Docker image.

Because the backend and compiled Angular frontend can be packaged together, the application can be deployed as a container.

A cloud deployment architecture can follow this pattern:

```text
Angular Frontend
       │
       ▼
Spring Boot Application
       │
       ▼
Docker Container
       │
       ▼
Container Registry
       │
       ▼
Cloud Container Service
```

For an AWS deployment, the Docker image could be stored in **Amazon Elastic Container Registry (ECR)** and deployed using a managed container service such as **Amazon ECS** or **Elastic Beanstalk**. The application's port 8080 would then be exposed through the appropriate AWS networking configuration.

---

## Key Concepts Demonstrated

This project demonstrates several important Java and backend-development concepts:

* Object-oriented programming
* Spring Boot
* REST API development
* Dependency injection
* Spring Data JPA
* Hibernate ORM
* Entity relationships
* Repository pattern
* Service layer architecture
* DTO/request/response models
* Object conversion
* Date handling with `LocalDate`
* Database persistence
* Angular frontend integration
* Maven build automation
* Docker containerization
* Cloud deployment concepts

## Project Structure

```text
d387-advanced-java-main/
│
├── pom.xml
│
└── src/
    └── main/
        ├── java/
        │   └── edu/wgu/d387_sample_code/
        │       ├── convertor/
        │       │   ├── ReservationRequestToReservationEntityConverter.java
        │       │   ├── ReservationService.java
        │       │   ├── ReservationServiceImpl.java
        │       │   ├── RoomEntityToReservableRoomResponseConverter.java
        │       │   ├── RoomService.java
        │       │   └── RoomServiceImpl.java
        │       │
        │       ├── entity/
        │       │   ├── ReservationEntity.java
        │       │   └── RoomEntity.java
        │       │
        │       ├── model/
        │       │   ├── Links.java
        │       │   ├── Self.java
        │       │   ├── request/
        │       │   │   └── ReservationRequest.java
        │       │   └── response/
        │       │       ├── ReservableRoomResponse.java
        │       │       └── ReservationResponse.java
        │       │
        │       ├── repository/
        │       │   ├── ReservationRepository.java
        │       │   └── RoomRepository.java
        │       │
        │       └── rest/
        │           ├── ReservationResource.java
        │           └── ResourceConstants.java
        │
        └── resources/
            ├── application.properties
            └── static/
                └── Angular production build
```

## What I Learned

Working on this project provided practical experience building a full-stack application with Java and Spring. The project demonstrates how a frontend can communicate with a REST API, how Spring Data JPA abstracts database operations, and how entities can be transformed into API-specific request and response objects.

The project also provided experience with containerization and cloud deployment concepts, including packaging a Spring Boot application into a Docker image and planning its deployment to a managed cloud container platform.
