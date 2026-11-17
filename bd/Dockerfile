FROM mongo:latest
ENV MONGO_INITDB_DATABASE=Biblioteca
ENV MONGO_INITDB_ROOT_USERNAME=root
ENV MONGO_INITDB_ROOT_PASSWORD=root

VOLUME /data/db
COPY ./init-mongo.js /docker-entrypoint-initdb.d/

EXPOSE 27017