# Only a Java runtime (JRE) is needed to run, not the full JDK
FROM eclipse-temurin:21-jre-alpine
# Copy the compiled code into the image
COPY Main.class .
# Run it
CMD ["java", "Main"]