FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY target/*.jar app.jar
RUN addgroup -g 1000 javauser && adduser -u 1000 -G javauser -s /bin/false -D javauser && chown -R javauser:javauser /app
USER javauser
ENTRYPOINT ["java","-jar","app.jar"]