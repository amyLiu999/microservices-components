# microservices-components

* API Gateway 
* Eureka
* Hystrix


## Tutorial Video
https://m.youtube.com/watch?v=nElpCWmpSew&list=PLyHJZXNdCXsd2e3NMW9sZbto8RB5foBtp&index=1

Estimated time: 105 minutes;

## create database
```sql
CREATE DATABASE citizen_service;
CREATE DATABASE vaccination_center;
```

## Eureka Server Service
service name: EurekaServer
url: http://localhost:8761/

**remember to visit the url and check the Eureka page, you will see the service in that page**

### Dependency
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

### Configuration
```yaml
server:
  port:
    8761
eureka:
  client:
    fetch-registry: false
    register-with-eureka: false
```
### Code
Only need to add `@EnableEurekaServer`
```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaServerApplication.class, args);
	}

}
```

## Citizen Service
service name: CitizenService
### Dependency
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

### Configuration
define service id/name: spring.application.name = CITIZEN-SERVICE

This configuration will register this service to Eureka
```yml
server:
  port: 8081
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/citizen_service
    username: springstudent
    password: springstudent
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: update
    show=sql: true
    database-platform: org.hibernate.dialect.MySQL5InnoDBDialect

  application:
    name: CITIZEN-SERVICE
```

## VaccinationCenter service
service name：VaccinationCenter
### Dependency
```xml
<dependencies>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
    <version>2.2.10.RELEASE</version>
</dependency>
</dependencies>
```

### Configuration
define service id/name: spring.application.name = VACCINATION-CENTER

This configuration will register this service to Eureka
```yml
server:
  port: 8082
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/vaccination_center
    username: springstudent
    password: springstudent
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: update
    show=sql: true
    database-platform: org.hibernate.dialect.MySQL5InnoDBDialect

  application:
    name: VACCINATION-CENTER
```

### Code
#### Enable CircuitBreaker and LoadBalanced
```java
@SpringBootApplication
@EnableCircuitBreaker
public class VaccinationCenterApplication {

	public static void main(String[] args) {
		SpringApplication.run(VaccinationCenterApplication.class, args);
	}

	@Bean
	@LoadBalanced
	public RestTemplate getRestTemplate() {
		return new RestTemplate();
	}
}
```

#### Call another service
```java
@RestController
@RequestMapping("/vaccinationcenter")
public class VaccinationCenterController {
    @Autowired
    private RestTemplate restTemplate;

    @GetMapping("/id/{id}")
    @HystrixCommand(fallbackMethod = "handleCitizenDownTime")
    public ResponseEntity<RequiredResponse> getAllDataBasedonCenterId(@PathVariable Integer id) {

        RequiredResponse response = new RequiredResponse();
        //get vaccination center detail
        VaccinationCenter center = centerRepository.findById(id).get();
        response.setCenter(center);

        //then get all citizen registered in that center
        List<Citizen> listCitizen = restTemplate.getForObject("http://CITIZEN-SERVICE/citizen/id/" + id, List.class);
        response.setCitizens(listCitizen);
        return new ResponseEntity<RequiredResponse>(response, HttpStatus.OK);

    }

    public ResponseEntity<RequiredResponse> handleCitizenDownTime(@PathVariable Integer id) {

        RequiredResponse response = new RequiredResponse();
        //get vaccination center detail
        VaccinationCenter center = centerRepository.findById(id).get();
        response.setCenter(center);

        return new ResponseEntity<RequiredResponse>(response, HttpStatus.OK);
    }
}
```

* **`@HystrixCommand(fallbackMethod = "handleCitizenDownTime")`: when failed, then call handleCitizenDownTime method to handle;**
* `restTemplate.getForObject("http://CITIZEN-SERVICE/citizen/id/" + id, List.class);`
  * CITIZEN-SERVICE is the service id we registered to Eureka.
  * If we don't user Eureka, we need use real URL where the citizen service is hosted instead of CITIZEN-SERVICE
  * the host server may change, so the url may change, we have to change the source code to give the new url
    * from `restTemplate.getForObject("http://192.168.101:8080/citizen/id/" + id, List.class);`
    * to `restTemplate.getForObject("http://100.92.38:8088/citizen/id/" + id, List.class);`
  * With Eureka, if we change the host server, there is no need to change the source code.
    * `restTemplate.getForObject("http://CITIZEN-SERVICE/citizen/id/" + id, List.class);` still work, no need to change.
    * loose coupling.

## API Gateway
### Dependencies
```xml
<dependencies>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
</dependencies>
```
### Configurations
```yaml
spring:
  application:
    name: API_GATEWAY

  cloud:
    gateway:
      routes:
        - id: CITIZEN-SERVICE
          uri:
            lb://CITIZEN-SERVICE
          predicates:
            - Path=/citizen/**

        - id: VACCINATION-CENTER
          uri:
            lb://VACCINATION-CENTER
          predicates:
            - Path=/vaccinationcenter/**
```

