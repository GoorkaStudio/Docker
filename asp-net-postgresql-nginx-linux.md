# Asp.Net Core (.Net 6) web application with PostgreSQL and Nginx


Create folder structure from https://github.com/GoorkaStudio/Docker/blob/main/asp-net-nginx-linux-simple.md

# Postgres
1. Create directory `mkdir postgres` and go to directory `cd postgres/`
2. Create new file Dockerfile `touch Dockerfile` 

```
version: "3.7"

services:
  postgres-service:
    container_name: postgres
    image: "postgres"
#    ports:
#      - 5432:5432
    volumes:
      - ./app-data/postgres:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: angielskiAnkieta
      POSTGRES_PASSWORD: angielskiAnkieta
    restart: always
#    build:
#      context: ./postgres
#      dockerfile: ./Dockerfile

  asptest-service:
    depends_on:
      - postgres-service
    container_name: asptest
    image: asptest
    restart: always
    build:
      context: ./asp-www

  nginx:
    container_name: nginx-main
    depends_on:
      - asptest-service
    ports:
      - 80:80
    image: nginx-test
    restart: always
    build:
      context: ./nginx
#      dockerfile: .Dockerfile

```
