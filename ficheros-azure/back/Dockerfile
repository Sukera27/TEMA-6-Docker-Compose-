FROM eclipse-temurin:17-jre

WORKDIR /app
COPY pokeAPI-0.0.1-SNAPSHOT.jar app.jar

# Debe coincidir con server.port de application.yaml
EXPOSE 8085

ENTRYPOINT ["java", "-jar", "app.jar"]
