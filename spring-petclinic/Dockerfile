FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY target/*.jar app.jar
RUN useradd -s /bin/false -u 1000 javauser && chown -R javauser:javauser /app
USER javauser
ENTRYPOINT ["java","-jar","app.jar"]