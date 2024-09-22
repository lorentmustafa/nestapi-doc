# Microservice Design Documentation
## Project Overview


This project is a microservice designed using **TypeScript** and **NestJS** for handling data operations. It supports data storage and retrieval via `POST /data` and `GET /data` endpoints. The architecture is designed for scalability, maintainability, and high performance, utilizing **CQRS**, **Sagas**, and **caching** strategies to handle large datasets efficiently.

## Table of Contents
1. [Technologies Used](#technologies-used)
2. [API Design](#api-design)
    - [POST /data](#post-data)
    - [GET /data](#get-data)
    - [Input Validation](#input-validation)
3. [Architecture Overview](#architecture-overview)
    - [Why CQRS and Sagas in NestJS?](#why-cqrs-and-sagas-in-nestjs)
    - [Caching Strategy](#caching-strategy)
    - [Database](#database)
    - [ORM Choice](#orm-choice)
    - [Folder Structure and Principles](#folder-structure-and-principles)
4. [Containerization](#containerization)
    - [Dockerfile](#dockerfile)
    - [Docker Compose](#docker-compose)
5. [Kubernetes Deployment](#kubernetes-deployment)
    - [Kubernetes Manifest Files](#kubernetes-manifest-files)
6. [CI/CD Pipeline](#cicd-pipeline)
    - [GitHub Actions](#github-actions)
7. [Testing Strategy](#testing-strategy)
    - [Unit Tests with Jest](#unit-tests-with-jest)
8. [Scaling Strategy for the NestJS Application](#scaling-strategy-for-the-nestjs-application)
    - [Horizontal Scaling on Kubernetes](#horizontal-scaling-on-kubernetes)
    - [PostgreSQL Scaling Strategies](#postgresql-scaling-strategies)
    - [Caching Strategy](#caching-strategy)
9. [Project Diagram](#project-diagram)

---

## Technologies Used

- **Language**: TypeScript
- **Framework**: NestJS
- **Database**: PostgreSQL for persistent storage, Redis for caching
- **ORM**: TypeORM or Prisma
- **Caching**: In-memory cache, Redis
- **API Documentation**: OpenAPI Swagger

---

## API Design

### POST /data

- Accepts JSON data, validates the input, and stores it in PostgreSQL.
- Example payload:
    ```json
    {
        "name": "name",
        "description": "description"
    }
    ```

### GET /data

- Retrieves data from the system, either from the cache (if available) or directly from the database.
- Optional parameters with default values will be provided on the endpoint, so the endpoint will be performant on large datasets


### Input Validation

  -Input validation is crucial in any application to ensure that the data received from users or other systems is in the correct format and type. NestJS provides a powerful and flexible mechanism for input validation through **DTOs (Data Transfer Objects)** and **Pipes**. The validation is mainly handled by the **`class-validator`** and **`class-transformer`** packages, which integrate seamlessly with NestJS.

## DTO (Data Transfer Object)

In NestJS, DTOs are used to define the shape of the input data. They are simple TypeScript classes where each property represents a field that the API will accept. 

By combining DTOs with decorators from the `class-validator` package, we can ensure that the incoming data is validated before it reaches the controller or service.

### Example: UserDTO with Validation

```typescript
// src/users/dto/create-user.dto.ts

import { IsString, IsEmail, IsNotEmpty, MinLength } from 'class-validator';

export class CreateUserDto {
  @IsString()
  @IsNotEmpty({ message: 'Name is required' })
  readonly name: string;

  @IsEmail({}, { message: 'Invalid email address' })
  readonly email: string;

  @IsString()
  @MinLength(8, { message: 'Password must be at least 8 characters long' })
  readonly password: string;
}
```

The ValidationPipe is responsible for triggering validation based on the decorators defined in the DTO class. When the ValidationPipe is used, it validates the incoming data against the DTO.

### Setting up Validation Pipe globally for request validation

```typescript
import { ValidationPipe } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(3000);
}
bootstrap();

```

---

## Architecture Overview

# Folder Structure and Principles

The project follows a **modular architecture** approach, breaking down functionality into individual modules. This structure provides better scalability, maintainability, and testability. Each module encapsulates its own logic and is responsible for handling a specific domain of the application.

## Folder Structure


```plaintext
/src
│
├── /modules
│   ├── /data
│   │   ├── /application
│   │   │   ├── /commands
│   │   │   │   ├── create-data.command.ts
│   │   │   │   └── handlers
│   │   │   │       └── create-data.handler.ts
│   │   │   ├── /queries
│   │   │   │   ├── get-data.query.ts
│   │   │   │   └── handlers
│   │   │   │       └── get-data.handler.ts
│   │   │   └── /sagas
│   │   │       └── data.saga.ts
│   │   │
│   │   ├── /domain
│   │   │   ├── /entities
│   │   │   │   └── data.entity.ts
│   │   │   └── /repositories
│   │   │       └── data.repository.ts
│   │   │
│   │   ├── /infrastructure
│   │   │   ├── /database
│   │   │   │   └── postgres.provider.ts
│   │   │   └── /cache
│   │   │       └── redis.provider.ts
│   │   │
│   │   ├── data.controller.ts
│   │   ├── data.module.ts
│   │   └── data.service.ts
│   │
│   └── /shared
│       └── cache.service.ts
│
├── /config
│   ├── app.config.ts
│   └── database.config.ts
│
├── /common
│   ├── /exceptions/
│   ├── /filters/
│   ├── /interceptors/
│   └── /pipes/
│
├── app.module.ts
└── main.ts
```
## Explanation of Folder Structure

### Modules
- **/modules**: Contains all application modules, promoting a modular structure that enhances code organization and separation of concerns.

### Data Module
- **/data**: Contains all components related to the data domain.
  - **/application**: Contains business logic and application workflows.
    - **/commands**: Handles commands that modify data.
      - **create-data.command.ts**: Represents the command to create new data.
      - **handlers**: Contains command handlers like `create-data.handler.ts` that execute the command logic.
    - **/queries**: Handles queries to retrieve data.
      - **get-data.query.ts**: Represents the query for fetching data.
      - **handlers**: Contains query handlers like `get-data.handler.ts` that execute the query logic.
    - **/sagas**: Manages long-running processes and side effects.
  - **/domain**: Represents the core business logic and entities.
    - **/entities**: Defines the structure of data models, such as `data.entity.ts`.
    - **/repositories**: Defines repository interfaces, such as `data.repository.ts`, for data access.
  - **/infrastructure**: Contains implementation details like database providers.
    - **/database**: Contains the database provider, such as `postgres.provider.ts`.
    - **/cache**: Contains cache providers like `redis.provider.ts`.
  - **data.controller.ts**: Manages HTTP requests and responses.
  - **data.module.ts**: The module definition that bundles related components together.
  - **data.service.ts**: Contains business logic and service methods for data handling.

### Shared
- **/shared**: Contains shared services used across the application, such as caching services.

### Config
- **/config**: Configuration files for the application.

### Common
- **/common**: Contains shared functionalities like custom exceptions, filters, interceptors, and pipes.

### Application Entry Points
- **app.module.ts**: The root module that imports all necessary modules.
- **main.ts**: The entry point for the application.

## Benefits of the Modular and Clean Architecture Approach

1. **Separation of Concerns**: Each module focuses on a specific domain or functionality, making it easier to manage and reason about the code.

2. **Scalability**: The architecture can grow as the application expands. New modules can be added without affecting existing ones, allowing teams to work on different features simultaneously.

3. **Maintainability**: With a clear structure, developers can quickly locate and modify parts of the application. This reduces the time required for updates and debugging.

4. **Testability**: Isolated modules allow for easier unit testing and integration testing. Each component can be tested independently, ensuring that changes do not break functionality.

5. **Reusability**: Shared components, such as services and utilities, can be reused across different modules, reducing code duplication.

6. **Improved Collaboration**: Teams can work on different modules without stepping on each other's toes, facilitating better collaboration and parallel development.

7. **Enhanced Performance**: With CQRS and caching strategies in place, the architecture can handle high data volumes efficiently while maintaining performance.

This modular and clean architecture is particularly beneficial for large-scale applications, as it promotes a structured approach to development, making it easier to handle complexity and growth.


### Why CQRS and Sagas in NestJS?

The architecture uses **CQRS** (Command Query Responsibility Segregation) to separate write and read operations. This allows:
- **Scalability**: Queries and commands can scale independently.
- **Optimization**: Query replicas can be used to offload read traffic, while commands can target master instances for writes.
  
**Sagas** are used in this project to orchestrate processes after commands are executed, such as updating the Redis cache in the background after data is written to PostgreSQL.

### Data Caching

The system uses a **3-tier caching strategy**:
1. **In-memory cache** (e.g., 30 seconds): Fastest access.
2. **Redis cache** (e.g., 1 hour): Secondary fallback.
3. **PostgreSQL**: Last fallback if data isn't found in memory or Redis.

This approach reduces the load on the database and improves overall performance.

### Database

The application uses **PostgreSQL** as the primary database because:
- **Relational data** can be queried using complex joins and links.
- It supports high-performance scaling for read replicas.

### ORM Choice

You can use either **TypeORM** or **Prisma** for database access:
- **TypeORM**: Natively supported by NestJS, works well with PostgreSQL, and has strong typing support.
- **Prisma**: Offers a clean, modern API for querying databases, and may be easier for complex queries.

---

## Containerization

### Dockerfile

The project should have a `Dockerfile` to containerize the application. The basic steps are:
1. **Use Node.js base image**: Start with an official Node.js image.
2. **Install dependencies**: Copy the `package.json` and install project dependencies.
3. **Copy source code**: Copy the entire source code into the container.
4. **Expose ports**: Ensure the right ports are exposed.
5. **Run the app**: Set the entry point to run the NestJS app.

```Dockerfile
# Base image
FROM node:16

# Set working directory
WORKDIR /app

# Install dependencies
COPY package.json ./
RUN npm install

# Copy source code
COPY . .

# Build the application
RUN npm run build

# Expose application port
EXPOSE 3000

# Start the app
CMD ["npm", "run", "start:prod"]
```

After the docker file is created you can build the image using :
docker build -t my-app .

After the build is successful, you can run the container by using:
docker run -p 3000:3000 my-app


### Docker Compose
A docker-compose.yml can be used for development purposes if multiple services like Redis and PostgreSQL need to run locally.

```Dockerfile
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - '3000:3000'
    environment:
      DATABASE_URL: postgres://user:pass@postgres:5432/mydb
      REDIS_URL: redis://redis:6379
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:12
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    ports:
      - '5432:5432'

  redis:
    image: redis:alpine
    ports:
      - '6379:6379'

```

Running docker-compose up will spin up the application, the Postgres database, and Redis


### Kubernetes deployment

## Kubernetes manifest files

The application will be deployed in a Kubernetes cluster.
Below are some examples on how the manifest files could look like:

### Deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nestjs-app-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nestjs-app
  template:
    metadata:
      labels:
        app: nestjs-app
    spec:
      containers:
        - name: nestjs-app
          image: myrepo/nestjs-app:${GITHUB_RUN_ID} # Dynamic image using GitHub Actions run ID
          ports:
            - containerPort: 3000
          env:
            - name: NODE_ENV
              value: "production"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DATABASE_URL

```

### Service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nestjs-app-service
spec:
  selector:
    app: nestjs-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: ClusterIP
```

### Ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nestjs-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nestjs-app-service
                port:
                  number: 80

```

The PostgreSQL and Redis services would also be deployed within the cluster for low-latency access.


### CI/CD Pipeline

## Github Actions

A GitHub Actions pipeline can be used for CI/CD to automate:

- **Code Integration**: Running linting and basic tests on each pull request
- **Testing**: Running unit tests automatically using Jest
- **Build & Deploy**: Building the Docker image and deploying it to the Kubernetes cluster

### Example:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
      - develop
  pull_request:
    branches:
      - main
      - develop

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Set up Node.js
        uses: actions/setup-node@v2
        with:
          node-version: '16'

      - name: Install dependencies
        run: npm install

      - name: Run linting
        run: npm run lint

  test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Set up Node.js
        uses: actions/setup-node@v2
        with:
          node-version: '16'

      - name: Install dependencies
        run: npm install

      - name: Run tests
        run: npm run test

      - name: Generate coverage report
        run: npm run coverage

      - name: Check coverage
        run: |
          if [ $(npm run coverage --silent | grep -o '[0-9]*' | head -n 1) -lt 80 ]; then
            echo "Coverage is below 80%"
            exit 1
          fi

  deploy:
    runs-on: ubuntu-latest
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Set up Kubernetes CLI
        uses: azure/setup-kubectl@v1
        with:
          version: 'latest'

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1

      - name: Log in to Docker Hub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build Docker image
        run: |
          docker build -t my-app:${{ github.run_id }} .

      - name: Push Docker image
        run: |
          docker push my-app:${{ github.run_id }}

      - name: Set Kubernetes context
        run: |
          echo "${{ secrets.KUBECONFIG }}" > kubeconfig
          kubectl config set-context --current --kubeconfig=kubeconfig

      - name: Run database migrations
        run: |
          kubectl exec -it deployment/my-app -- npm run migrate

      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/my-app my-app=my-app:${{ github.run_id }} --record
          kubectl rollout status deployment/my-app

      - name: Notify deployment success
        run: |
          echo "Deployment to Kubernetes successful!"

```


With this we make sure that linting and tests are run on pull requests to main, making sure we do not break anything before merging to the branch. Making use which branch is triggering the workflow, we will then build the Docker image and Deploy to Kubernetes.

A github.run_id approach is used so every deployment will have a unique identifier of the image running on Production/Development environment. This can come in handy on times when a revert is needed to be made after the successful deploy.

## Future considerations

Implementing a notification system for failures and successes using tools like Slack or email could benefit faster responsitivity.
Caching strategies for npm, to make builds faster


### Testing Strategy

### Unit tests with Jest

Unit testing is done using Jest, a widely used testing framework in the Node.js ecosystem. Tests will be written for:

##
- **Commands**: Ensure correct data processing
- **Queries**: Validate that queries return correct data
- **Sagas**: Ensure background processes like cache updates are triggered correctly

Example test case:
```Typescript
describe('CreateDataCommandHandler', () => {
  it('should create new data and trigger the saga', async () => {
    // Arrange
    const dto = { name: 'Test', description: 'Test description' };
    const command = new CreateDataCommand(dto);
    const handler = new CreateDataCommandHandler(mockRepo);

    // Act
    await handler.execute(command);

    // Assert
    expect(mockRepo.create).toHaveBeenCalledWith(dto);
    expect(mockSaga).toHaveBeenTriggered();
  });
});

```


# Scaling Strategy for the NestJS Application

## Horizontal Scaling on Kubernetes
The application will utilize Kubernetes for horizontal scaling, allowing it to dynamically adjust to varying traffic loads. The key features of this approach include:

### Horizontal Pod Autoscaler (HPA)
- **Definition**: The HPA automatically adjusts the number of pods in a deployment based on observed metrics such as CPU utilization or memory usage.
- **Configuration**: Metrics will be monitored, and thresholds will be defined to trigger scaling events. For instance, if CPU usage exceeds 70% for a defined duration, additional pods will be deployed to distribute the load.
- **Benefits**: This ensures high availability and responsiveness, especially during peak traffic periods, without manual intervention.

## PostgreSQL Scaling Strategies
To manage billions of rows efficiently, PostgreSQL scaling will employ several strategies:

### 1. Indexing
- **Purpose**: Indexes enhance query performance by allowing the database to find rows faster.
- **Implementation**: Primary keys and commonly queried fields (e.g., `created_at`, `user_id`) will be indexed.
- **Effectiveness**: Proper indexing can reduce query times from seconds to milliseconds, significantly improving the performance of read operations.

### 2. Partitioning
- **Concept**: Partitioning divides large tables into smaller, more manageable segments based on a specific criterion.
- **Implementation**: 
  - **Time-based Partitioning**: For instance, data can be partitioned by month or year, making it easier to manage and query.
  - **Example**: A table of user activity logs could be partitioned such that logs from 2022 are in one partition and logs from 2023 in another.
- **Benefits**: This approach improves query performance by allowing the database to scan only relevant partitions, reducing the amount of data processed during queries.

### 3. Connection Pooling
- **Mechanism**: Connection pooling maintains a pool of active database connections that can be reused, reducing the overhead of establishing new connections.
- **Configuration**: A connection pooler like PgBouncer will be employed to manage the number of active connections.
- **Advantage**: This strategy increases efficiency, particularly under high load, by minimizing latency and resource consumption.

## Caching Strategy
To optimize performance further and reduce database load, we will utilize caching effectively:

### Write-Through Cache with Lazy Loading
- **Write-Through Cache**: When new records are inserted through the `POST /data` endpoint, the cache will be updated immediately. The latest data will be written to both the PostgreSQL database and the Redis cache, ensuring data consistency.
The logic will be handled by making use of saga's, hereby removing the logic from the handlers.
- **Lazy Loading**: When fetching data, the application will first check the cache. If the requested data is not found in the cache, it will then query the PostgreSQL database. This strategy minimizes database reads, as frequently accessed data will be served from the cache.

## Conclusion
By adopting these comprehensive strategies—leveraging Kubernetes for horizontal scaling, employing effective indexing and partitioning in PostgreSQL, utilizing connection pooling, and implementing a robust cache management system through sagas—the application will be well-prepared to efficiently handle billions of rows while maintaining high performance and responsiveness.

This approach not only optimizes resource utilization but also enhances the user experience by ensuring fast response times, even under heavy load.

### Project Diagram

![Alt text](diagram.png)