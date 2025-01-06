# Backend documentation
## technical stack

* Language: java
* Framework: Quarkus
* Database: PostgreSQL

## Requirements

* Java 21 (AWS Corretto for example)
* Docker or Podman
* Maven

## Installation
### Maven and Java
You can use SDKMAN to install Java and Maven: [https://sdkman.io/](https://sdkman.io/)

### Docker or Podman
You can install Docker using the following link: [https://docs.docker.com/get-docker/](https://docs.docker.com/get-docker/)
You can install Podman using the following link: [https://podman.io](https://podman.io)

### PostgreSQL
Use docker to run a PostgreSQL instance:
```shell
docker run -it --rm=true --name quarkus_test -e POSTGRES_USER=quarkus_test -e POSTGRES_PASSWORD=quarkus_test -e POSTGRES_DB=quarkus_test -p 5432:5432 postgres:13.3
```

