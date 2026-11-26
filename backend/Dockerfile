# Image d'exécution légère : le jar est produit par l'étape Build de Jenkins
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

# Utilisateur non-root
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring

COPY target/backend-1.0.0.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
