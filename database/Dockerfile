# Start from the official Postgres image
FROM postgres:17.2-alpine

# Any .sql file in this folder runs automatically the first time the DB starts
COPY sql/ /docker-entrypoint-initdb.d/