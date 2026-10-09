# ---- Stage 1: BUILD (has JDK + Maven, big image) ----
FROM eclipse-temurin:21-jdk-alpine AS myapp-build
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME
# Install Maven
RUN apk add --no-cache maven
# Copy the project description, then the code
COPY pom.xml .
COPY src ./src
# Compile and package into a .jar (skip tests to go faster)
RUN mvn package -DskipTests

# ---- Stage 2: RUN (only JRE, small image) ----
FROM eclipse-temurin:21-jre-alpine
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME
# Take only the .jar from stage 1, leave all the build tools behind
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
# Start the app
ENTRYPOINT ["java", "-jar", "myapp.jar"]