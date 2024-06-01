# Using Redis with SpringBoot

On my other [project](test-redis-with-generated-data.md), I use Redis as a caching layer. This time I would like to use it as a database in a SprintBoot project.

## Bring up a local Redis with docker

``` bash
docker run --name my-redis -p 6379:6379 -d redis
```

## Redis Client

There are two popular clients - `Lettuce` and `Jedis`. I picked Jedis this time.

``` xml
<!-- pom.xml -->
<dependency>
  <groupId>redis.clients</groupId>
  <artifactId>jedis</artifactId>
</dependency>
```

Local development environment setup:

``` bash
# application.properties

spring.redis.host=localhost
spring.redis.port=6379
```

Sample configuration class

``` java
// configs/RedisConfig.java

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.jedis.JedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericToStringSerializer;

@Configuration
public class RedisConfig {

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        return new JedisConnectionFactory();
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory());
        template.setValueSerializer(new GenericToStringSerializer<>(Object.class));
        return template;
    }
}
```

Sample model class

``` java
package com.tnite.redistest;

import lombok.Data;
import org.springframework.data.annotation.Id;
import org.springframework.data.redis.core.RedisHash;

import java.io.Serializable;

@Data
@RedisHash("Employee")
public class Employee implements Serializable {

    @Id
    private Long id;
    private String name;
    private Double salary;
    ...
}
```

Everything was going smoothly until I realized that the ElastiCache instance is SSL-enabled and password-protected.


## Make it works with Elasticache

I want to retrieve all configuration parameters from environment variables. To ensure compatibility with the local development setup, the following code can handle configurations without a password and with SSL disabled.

``` java
@Slf4j
@Configuration
public class RedisConfig {

    @Value("${spring.redis.host:localhost}")
    private String host;

    @Value("${spring.redis.port:6379}")
    private Integer port;

    @Value("${spring.redis.password:@null}")
    private String password;

    @Value("${spring.redis.ssl.enabled:false}")
    private Boolean useSsl;

    @Bean
    RedisConnectionFactory redisConnectionFactory() {

        JedisConnectionFactory factory;
        JedisClientConfiguration config = JedisClientConfiguration.builder().build();
        if (useSsl) {
            log.info("Redis with SSL enabled!");
            config = JedisClientConfiguration.builder().useSsl().build();
        }

        RedisStandaloneConfiguration redisConfig = new RedisStandaloneConfiguration(host, port);

        if (password != null && !password.isEmpty()) {
            log.info("Redis with password is used!");
            redisConfig.setPassword(password);
        }

        factory = new JedisConnectionFactory(redisConfig, config);

        return factory;
    }

}
```