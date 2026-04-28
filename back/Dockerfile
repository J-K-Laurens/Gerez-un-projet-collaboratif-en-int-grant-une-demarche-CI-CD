FROM maven:3.9-eclipse-temurin-17-alpine AS build
RUN mkdir -p /workspace
WORKDIR /workspace
COPY pom.xml /workspace
COPY src /workspace/src
RUN mvn -B -f pom.xml clean package -DskipTests
FROM eclipse-temurin:17-jdk-alpine
COPY --from=build /workspace/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar","app.jar"]
