# Tinsa Service Discovery: Eureka

##What is service discovery, how does it work

Is a cloud pattern where clients register within a server, this allows that communication within services doesn't happen directly between them but via service discovery server, delegating the location of all these services to service discovery server.
A service only needs to know where the service discovery server is, and ask it for other services just using their names.
For that purpose each service defines a name which will be used for registering and for being accessed by other services: eureka.client.name

Eureka is just the Netflix implementation for this pattern but there are other alternatives, like Consul or Zookeeper.

###Registry process:

 - Client sends a heartbeat to the server, two things can happen:
	 - Service is already registered, then the heartbeat means the service is still alive.
	 - Service is not registered, the server returns an unknown service error which forces the service to register.
 - The Service sends a register request to the server, which includes the service within its register.
 - Server updates its cache with the new service information.
 - The load balancer is updated.

###Self Preservation Mode and service eviction
Eureka defines a minimun threshold percentage, 85% of heartbeats expected to come from all services registered eureka.server.renewsthreshold.
When total heartbeats surpass below this threshold the self preservation mode is activated and the server stops to evict services not sending heartbeats from registry.
This mode is used to prevent temporal network problems.

###Useful properties:

####Service registering in Eureka properties

```properties
eureka.instance.leaseRenewalIntervalInSeconds
#Heartbeats emission frequency, it can be used to speed up registration process but it can also break renews threshold calculation, by default is 30

eureka.client.initialInstanceInfoReplicationIntervalSeconds
#Initial delay before trying to register, also used to speed up registration process

eureka.client.registryFetchIntervalSeconds
#Frequency for fetching Eureka server registry information, 30 by default

eureka.client.serviceUrl.defaultZone
#Location of eureka server
```

####Eureka Server properties
```properties
eureka.instance.leaseExpirationDurationInSeconds
#Waiting time before evicting a service which is not sending heartbeats, by default is 90, that is 3 heartbeats

eureka.client.fetchRegistry
#If set in eureka, indicates whether the client module of eureka server fetches information of its register or not.

eureka.client.registerWithEureka
#If set in eureka, indicates that the eureka server is registered with self or another eureka server or not.
#Eureka servers are usually registered in other servers for resilience purposes.
```

###Example of Eureka Server configuration:

- bootstrap.yml
```yml
server:
  port: 8761

spring:
  application:
    name: eureka-server
```

- application.yml
```yml
eureka:
  instance:
    leaseExpirationDurationInSeconds: 30
  client:
    fetchRegistry: true
    registerWithEureka: true
    serviceUrl:
      defaultZone: http://localhost:${server.port}/eureka/
  server:
    enableSelfPreservation: false
    waitTimeInMsWhenSyncEmpty: 15
```

##How to register a service in Eureka Server:

- pom.xml:

```xml
<dependencyManagement>
	...
	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-dependencies</artifactId>
			<version>Camden.SR3</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
	</dependencies>
	...
</dependencyManagement>
...
<dependencies>
	...
	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter-eureka</artifactId>
	</dependency>
	...
</dependencies>
```
- bootstrap.yml:

```yml
#Random free port configuration, suitable for cloud environments
eureka:
	server:
		port: 0

 #Application name used when registering service
 spring:
	 application:
	     name: rosetta
```

 - application.yml:

```yml
#Eureka server location
eureka:
	client:
		serviceUrl:
			defaultZone: http://eureka-server/eureka
```

 - Application.groovy

```groovy
@EnableDiscoveryClient

#For service within the same cloud access
@Bean
RestTemplate restTemplate() {
	return new RestTemplate()
}
```
 - Any class accessing a server within the cloud using discovery client and restTemplate

```groovy
@Autowired private DiscoveryClient discoveryClient

@Autowired private RestTemplate restTemplate

anymethod {
ServiceInstance serviceInstance = discoveryClient.getInstances('serviceName').first()

response = restTemplate.postForEntity(
    "${serviceInstance.uri}/endpoint",
    request,
    responseType
) }
```

##Glossary of tems
**Heartbeat**: State sharing purpose request. Services send heartbeats to the server indicating they are alive

##Documentation
[Eureka server config](https://github.com/Netflix/eureka/blob/master/eureka-core/src/main/java/com/netflix/eureka/EurekaServerConfig.java)
[Eureka instance config](https://github.com/spring-cloud/spring-cloud-netflix/blob/master/spring-cloud-netflix-eureka-client/src/main/java/org/springframework/cloud/netflix/eureka/EurekaInstanceConfigBean.java)
[Eureka client config](https://github.com/spring-cloud/spring-cloud-netflix/blob/master/spring-cloud-netflix-eureka-client/src/main/java/org/springframework/cloud/netflix/eureka/EurekaClientConfigBean.java)

[Eureka server endpoints](https://github.com/Netflix/eureka/wiki/Eureka-REST-operations)

[Spring io tutorial](https://spring.io/guides/gs/service-registration-and-discovery/)
[Paradigma digital Eureka tutorial part 1](https://www.paradigmadigital.com/dev/el-microservicio-eureka-a-fondo-12/)
[Paradigma digital Eureka tutorial part 2](https://www.paradigmadigital.com/dev/el-microservicio-eureka-a-fondo-22/)

##TODOs
TODO: Investigate the use of feign instead of restTemplate
TODO: Use Ribbon for load balance on top of services, instead of discoveryClient.getInstances('serviceName').first()
TODO: Investigate cloud access via Zuul
TODO: Client failover explanation in README