# TP DevOps

Rayane Karaouzene

The app has 4 parts: a PostgreSQL database, a Spring Boot API, a Vue.js front and an Apache httpd reverse proxy.

It is deployed here: http://rayane.karaouzene.takima.school

## Folders

- `database/` : database Dockerfile and SQL scripts
- `simple-api/` : backend API
- `httpd/` : Apache server and its config (`httpd.conf`)
- `front/` : front
- `java-hello/` : hello world test from TP1
- `docker-compose.yml` : runs everything locally
- `.github/workflows/` : GitHub Actions workflows
- `ansible/` : Ansible inventory, playbook and roles


# TP1 - Docker

### 1-1 For which reason is it better to run the container with a flag -e to give the environment variables rather than put them directly in the Dockerfile?

The Dockerfile is pushed on git, so a password written inside would be visible to everyone. With -e the password is only given when the container starts.

### 1-2 Why do we need a volume to be attached to our postgres container?

When a container is deleted its data is deleted too. The volume keeps the database data outside the container.

### 1-3 Document your database container essentials: commands and Dockerfile

The Dockerfile is in `database/Dockerfile`, it copies the SQL scripts from `database/sql/` that create and fill the tables. The container runs on the `app-network` network with the env variables and the `db-data` volume.

### 1-4 Why do we need a multistage build? And explain each step of this dockerfile.

We need the JDK and Maven to build the app but only the JRE to run it. In `simple-api/Dockerfile`, the first stage builds the .jar and the second stage only copies the .jar into a small JRE image.

### 1-5 Why do we need a reverse proxy?

So there is only one entry point (port 80) and the backend and database are not reachable from outside. The config is at the end of `httpd/httpd.conf`.

### 1-6 Why is docker-compose so important?

It describes all the containers in one file and starts them with one command, in the right order.

### 1-7 Document docker-compose most important commands

`docker compose up -d --build` to build and start, `docker compose ps` to see the containers, `docker compose logs -f` to see logs, `docker compose down` to stop and remove.

### 1-8 Document your docker-compose file

`docker-compose.yml` starts the database, backend and httpd on the same network. Only httpd has a port open, passwords come from a `.env` file that is not pushed, and the backend waits for the database to be ready.

### 1-9 Document your publication commands and published images in dockerhub

I used `docker tag` then `docker push`. My images are `raykrz/tp-devops-database`, `raykrz/tp-devops-simple-api`, `raykrz/tp-devops-httpd` and `raykrz/tp-devops-front`.

### 1-10 Why do we put our images into an online repo?

So the server or other people can download and run the same image without building it again.


# TP2 - GitHub Actions

There are 3 workflows in `.github/workflows/`: `test-backend.yml` runs the tests and SonarCloud, `build-push.yml` pushes the images if the tests passed on main, and `deploy.yml` deploys on the server if the push worked.

### 2-1 What are testcontainers?

A Java library that starts real Docker containers during tests. Here it starts a temporary Postgres database for the integration tests.

### 2-2 For what purpose do we need to use secured variables?

The workflow files are public in the repo, so passwords and tokens must be stored as GitHub secrets to stay hidden.

### 2-3 Why did we put needs: build-and-test-backend on this job? Maybe try without this and you will see!

Without it the jobs run at the same time, so images would be pushed even if the tests fail. I split the jobs into separate workflows, so I used `workflow_run` with a success check instead.

### 2-4 For what purpose do we need to push docker images?

So the server can download the tested image directly without needing the code.

### Quality gate

SonarCloud runs in `test-backend.yml` and uses the default "Sonar way" quality gate. The token is a GitHub secret.


# TP3 - Ansible

### 3-1 Document your inventory and base commands

The inventory is `ansible/inventories/setup.yml`, it has my server in the `prod` group, the `admin` user, the SSH key path and the variables used by the roles. I used `ansible all -i inventories/setup.yml -m ping` to test the connection and `-m setup` to get info about the server.

### 3-2 Document your playbook

`ansible/playbook.yml` runs the roles in order: docker, network, database, app, front, proxy. Each role is in `ansible/roles/<name>/tasks/main.yml`.

### 3-3 Document your docker_container tasks configuration

Each container has a name, an image from my Docker Hub, `pull: true` to get the newest image and is on `app-network`. The database also has env variables and a volume, the backend has the database env variables, and only httpd opens port 80.

### Is it really safe to deploy automatically every new image on the hub? explain. What can I do to make it more secure?

No, if someone gets my Docker Hub token they can push a bad image that goes directly to production. To make it safer we can ask for a manual approval before deploying, use version tags instead of `latest`, and encrypt the passwords with Ansible Vault.

### Front

The front is in `front/` and httpd sends `/api/` to the backend and the rest to the front (see `httpd/httpd.conf`). It is built and deployed by the same pipeline as the other images.
