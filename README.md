# TP DevOps – Part 01: Docker

3-tier application running in Docker:

- **HTTP server** (Apache httpd) – reverse proxy, the only entry point (port 80)
- **Backend API** (Spring Boot, Java 21)
- **Database** (PostgreSQL 17)

## Project structure

```
docker-tp/
├── database/
│   ├── Dockerfile
│   └── sql/
│       ├── 01-CreateScheme.sql
│       └── 02-InsertData.sql
├── java-hello/
│   ├── Dockerfile
│   └── Main.java
├── simple-api/
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/
├── httpd/
│   ├── Dockerfile
│   ├── httpd.conf
│   └── index.html
├── docker-compose.yml
├── .env            
└── README.md
```

One folder per image, each with its own `Dockerfile`.

---

## 1. Database

### Dockerfile

```dockerfile
# Official Postgres image (lightweight alpine version)
FROM postgres:17.2-alpine

# Every .sql file in /docker-entrypoint-initdb.d runs automatically,
# in alphabetical order, the first time the database starts
COPY sql/ /docker-entrypoint-initdb.d/
```

- `01-CreateScheme.sql` creates the `departments` and `students` tables.
- `02-InsertData.sql` inserts the initial data.
- The scripts only run when the data volume is empty (first start).

### Commands

```bash
# Create a network so containers can reach each other by name
docker network create app-network

# Build the image
docker build -t tp-database ./database

# Run the container
docker run -d \
  --name database \
  --network app-network \
  -e POSTGRES_DB=db \
  -e POSTGRES_USER=usr \
  -e POSTGRES_PASSWORD=pwd \
  -v db-data:/var/lib/postgresql/data \
  tp-database

# Adminer (web UI to inspect the database) on http://localhost:8090
docker run -d -p 8090:8080 --network app-network --name adminer adminer
```

| Flag | Meaning |
|---|---|
| `-d` | Run in the background (detached) |
| `--name` | Container name, also used as hostname on the network |
| `--network` | Attach the container to `app-network` |
| `-e` | Set an environment variable |
| `-v` | Mount a volume to persist data |
| `-p host:container` | Expose a container port on the host |

### 1-1 Why pass environment variables with `-e` instead of writing them in the Dockerfile?

The Dockerfile is committed to Git, so any password written in it is visible to anyone with access to the repository, and it gets baked into the image layers. Passing them with `-e` at runtime keeps secrets out of the code and the image. The same image can then also be reused in different environments (dev, prod) with different credentials.

### 1-2 Why do we need a volume attached to the Postgres container?

A container's filesystem is temporary: when the container is deleted, its data is lost. The volume `db-data` stores the database files (`/var/lib/postgresql/data`) outside the container, on the host. The data survives when the container is removed and recreated.

Test performed: `docker rm -f database`, then the same `docker run` again. The data was still present in Adminer.

### 1-3 Database container essentials

See the Dockerfile and commands above: create network → build image → run with env variables, network and volume.

---

## 2. Backend API

### 2.1 Java Hello World

```dockerfile
# Only a Java Runtime (JRE) is needed to run compiled code
FROM eclipse-temurin:21-jre-alpine
# Copy the compiled bytecode
COPY Main.class .
# Run it
CMD ["java", "Main"]
```

```bash
javac Main.java
docker build -t java-hello ./java-hello
docker run --rm java-hello      # prints "Hello World!"
```

### 2.2 Spring Boot API – multistage build

```dockerfile
# ---- Stage 1: BUILD ----
# JDK image: contains the compiler needed to build the app
FROM eclipse-temurin:21-jdk-alpine AS myapp-build
# Variable holding the app directory
ENV MYAPP_HOME=/opt/myapp
# Work inside this directory
WORKDIR $MYAPP_HOME
# Install Maven (build tool)
RUN apk add --no-cache maven
# Copy the project definition and the source code
COPY pom.xml .
COPY src ./src
# Compile and package the app into a .jar (tests skipped for speed)
RUN mvn package -DskipTests

# ---- Stage 2: RUN ----
# JRE image: only what is needed to run Java, much smaller
FROM eclipse-temurin:21-jre-alpine
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME
# Copy only the built .jar from stage 1
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
# Start the application
ENTRYPOINT ["java", "-jar", "myapp.jar"]
```

### 1-4 Why do we need a multistage build?

Building the app requires a full JDK and Maven, which are large, but running it only requires a JRE. With a multistage build:

- **Stage 1** uses the JDK and Maven to compile the code and produce the `.jar`.
- **Stage 2** starts from a small JRE image and only copies the `.jar`.

The final image is much lighter. It is also more secure, because it contains no compiler, no Maven and no source code. And the build is reproducible: nobody needs Java or Maven installed on their machine, Docker does everything.

### 2.3 Connection to the database

In `simple-api/src/main/resources/application.yml`:

```yaml
  datasource:
    url: jdbc:postgresql://${DB_HOST:database}:5432/${DB_NAME:db}
    username: ${DB_USER:usr}
    password: ${DB_PASSWORD:pwd}
```

Values come from environment variables, with defaults after the `:`. The hostname `database` works because both containers are on `app-network`, where Docker resolves container names automatically.

```bash
docker build -t tp-backend ./simple-api
docker run -d --name backend --network app-network -p 8080:8080 tp-backend
```

Test: `http://localhost:8080/departments/IRC/students` returns the students of the IRC department in JSON.

---

## 3. HTTP server

### Dockerfile

```dockerfile
# Official Apache image
FROM httpd:2.4
# Landing page
COPY index.html /usr/local/apache2/htdocs/
# Custom configuration (reverse proxy)
COPY httpd.conf /usr/local/apache2/conf/httpd.conf
```

### Getting the default configuration

```bash
docker cp httpd:/usr/local/apache2/conf/httpd.conf ./httpd/httpd.conf
```

### Reverse proxy configuration (added at the end of `httpd.conf`)

```apache
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so

<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass / http://backend:8080/
ProxyPassReverse / http://backend:8080/
</VirtualHost>
```

Every request on port 80 is forwarded to the `backend` container on port 8080.

### Useful commands

- `docker stats`: live CPU and memory usage of containers
- `docker inspect <container>`: full configuration of a container (network, IP, mounts…)
- `docker logs <container>`: output of the container

### 1-5 Why do we need a reverse proxy?

- **Single entry point:** only port 80 is exposed, while the backend and database stay hidden from the outside.
- **Security:** clients never talk directly to the backend.
- **Flexibility:** it can later handle HTTPS, load balancing across several backends, or serve a frontend, without changing the backend.

---

## 4. Docker Compose

### `.env` (not committed – listed in `.gitignore`)

```
POSTGRES_DB=db
POSTGRES_USER=usr
POSTGRES_PASSWORD=pwd
```

### `docker-compose.yml`

```yaml
services:
  database:
    build: ./database                 # build from database/Dockerfile
    container_name: database
    environment:                      # credentials read from .env
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - db-data:/var/lib/postgresql/data   # persist data
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:                      # checks Postgres is ready to accept connections
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      retries: 10
    # no ports exposed: only reachable inside app-network

  backend:
    build: ./simple-api
    container_name: backend
    environment:                      # injected into application.yml
      DB_HOST: database
      DB_NAME: ${POSTGRES_DB}
      DB_USER: ${POSTGRES_USER}
      DB_PASSWORD: ${POSTGRES_PASSWORD}
    networks:
      - app-network
    depends_on:
      database:
        condition: service_healthy    # start only once the DB is ready
    restart: unless-stopped
    # no ports exposed: only httpd talks to it

  httpd:
    build: ./httpd
    container_name: httpd
    ports:
      - "80:80"                       # the only port open to the host
    networks:
      - app-network
    depends_on:
      - backend
    restart: unless-stopped

networks:
  app-network:

volumes:
  db-data:
```

### 1-6 Why is docker-compose so important?

It describes the whole application (images, configuration, networks, volumes, dependencies) in one versioned file. One command starts, stops or rebuilds everything in the right order, instead of many manual `docker build` and `docker run` commands. That makes the setup reproducible for every developer and environment.

### 1-7 Most important docker-compose commands

| Command | Description |
|---|---|
| `docker compose up -d --build` | Build the images and start all services in the background |
| `docker compose ps` | List the running services |
| `docker compose logs -f <service>` | Follow the logs of a service |
| `docker compose stop` | Stop the services without removing them |
| `docker compose down` | Stop and remove the containers and network (volumes are kept) |
| `docker compose down -v` | Same, and also delete the volumes (data reset) |
| `docker compose config` | Validate and display the final configuration |

### 1-8 docker-compose file explained

- **Environment variables:** credentials are not hardcoded; they come from a `.env` file that is not committed.
- **Volume `db-data`:** the database data persists across restarts.
- **Network `app-network`:** services communicate by name (`database`, `backend`).
- **Ports:** only `httpd` exposes a port (80). The backend and database are not reachable from the host.
- **`depends_on` + `healthcheck`:** the backend waits until Postgres is actually ready, and httpd starts after the backend.
- **`restart: unless-stopped`:** containers restart automatically if they crash.

Result: `http://localhost/departments/IRC/students` works through the reverse proxy.

---

## 5. Publish

```bash
# Log in to Docker Hub
docker login

# Tag images with the Docker Hub username and a version
docker tag docker-tp-database USERNAME/tp-devops-database:1.0
docker tag docker-tp-backend  USERNAME/tp-devops-simple-api:1.0
docker tag docker-tp-httpd    USERNAME/tp-devops-httpd:1.0

# Push them to Docker Hub
docker push USERNAME/tp-devops-database:1.0
docker push USERNAME/tp-devops-simple-api:1.0
docker push USERNAME/tp-devops-httpd:1.0
```

### 1-9 Published images

| Image | Tag |
|---|---|
| `USERNAME/tp-devops-database` | 1.0 |
| `USERNAME/tp-devops-simple-api` | 1.0 |
| `USERNAME/tp-devops-httpd` | 1.0 |

### 1-10 Why do we put our images into an online repository?

So that other team members, servers or CI/CD pipelines can download (`docker pull`) and run exactly the same image without rebuilding it from source. It also keeps versioned history through tags (1.0, 1.1…), which makes it possible to deploy or roll back to a specific version.
