FROM maven:3.8.6-openjdk-11 AS build
WORKDIR /build

COPY pom.xml .
RUN mvn -N install

COPY microservice-kubernetes-demo-order/pom.xml microservice-kubernetes-demo-order/pom.xml
WORKDIR /build/microservice-kubernetes-demo-order
RUN mvn dependency:go-offline -B

COPY microservice-kubernetes-demo-order/src ./src
RUN mvn clean package -DskipTests

FROM openjdk:11.0.2-jre-slim
WORKDIR /app
COPY --from=build /build/microservice-kubernetes-demo-order/target/microservice-kubernetes-demo-order-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-Xmx400m", "-Xms400m", "-jar", "app.jar"]
