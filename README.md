# BobApp

## Front-end

### Local
```
cd front
npm install
npm run start
```

### Docker
```
cd front
docker build -t bobapp-front .
docker run -p 8080:8080 --name bobapp-front -d bobapp-front
```

## Back-end

### Local
```
cd back
mvn clean install
mvn spring-boot:run
```

### Tests
```
mvn clean install
```

### Docker
```
cd back
docker build -t bobapp-back .
docker run -p 8080:8080 --name bobapp-back -d bobapp-back
``` 