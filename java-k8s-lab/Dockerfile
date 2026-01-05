# --------- Build Stage ---------
FROM adoptopenjdk:11 as build
WORKDIR /app

# Copy the Java source file into the container
COPY HelloController.java .

# Compile the Java source file
RUN javac HelloController.java

# Package all compiled classes (including inner classes) into an executable jar
RUN jar cfe app.jar HelloController *.class

# --------- Run Stage ---------
FROM adoptopenjdk:11-jre-hotspot
WORKDIR /app

# Copy the executable jar from the build stage
COPY --from=build /app/app.jar .

# Expose the port on which the app listens
EXPOSE 8080

# Run the jar file
ENTRYPOINT ["java", "-jar", "app.jar"]
