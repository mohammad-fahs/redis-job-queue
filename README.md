# Introduction

A job queue is a data structure storing *execution* requests. Job dispatchers submit the tasks they want to execute in that data structure. On the other side, job consumers poll the requests and execute them.
→ how it works ? 

- Users submit programs, or "jobs," that they want the computer to run.
- These jobs are added to the queue.
- A scheduler program manages the queue, prioritizing and selecting jobs to run based on various factors like urgency or resource requirements.

![Untitled](https://github.com/mohammad-fahs/redis-job-queue/assets/109221274/6724f84c-91f2-4512-946b-28a094ea37bd)


### Redis and Job Queue

Redis is a popular in-memory data structure store that can be used to create a job queue due to its support for data structures like lists, sets, and sorted sets, as well as its ability to handle pub/sub (publish/subscribe) messaging.

→ the most common data structured Redis offer that can be used in Job Queue are 

- **Lists:** You can use Redis lists to store jobs as items in the list. New jobs can be added to the end of the list (enqueue), and workers can fetch jobs from the beginning of the list (dequeue).
- **Sorted Sets:** Another approach is to use sorted sets, where each job has a score representing its priority or timestamp. Workers can fetch jobs based on their score, allowing for priority-based processing or time-based scheduling.

# Redis Run on Docker

To begin, we'll use Docker Compose to set up Redis. We've created a YAML configuration file that defines two services: Redis and Redis Insight.

Here's a breakdown of the YAML file:

- **Version:** We're using version 3.3 of Docker Compose.
- **Services:**
    - **Redis:** This service is based on the Redis 6.0.7 image. It's named "`redis`" and set to restart always. We've also defined a volume named "`redis_volume_data`" to persist Redis data and mapped port 6379 on the host to port 6379 in the container for Redis access.
    - **Redis Insight:** This service uses the latest Redis Insight image from Redis Labs. It's named "`redis_insight`," set to restart always, and mapped port 8001 on the host to port 8001 in the container for Redis Insight access. Additionally, we've defined a volume named "`redis_insight_volume_data`" to persist Redis Insight data.
- **Volumes:** We've specified two volumes:
    - **`redis_volume_data`:** This volume is used by the Redis service to store its data persistently.
    - **`redis_insight_volume_data`:** This volume is used by the Redis Insight service to store its data persistently.

By running this Docker Compose configuration, we'll have Redis up and running, accessible via port 6379, and Redis Insight accessible via port 8001. The volumes ensure that data is persisted even if the containers are stopped or restarted.

```yaml
version: "3.3"
services:
  redis:
    image: redis:6.0.7
    container_name: redis
    restart: always
    volumes:
      - redis_volume_data:/data
    ports:
      - 6379:6379
  redis_insight:
    image: redislabs/redisinsight:latest
    container_name: redis_insight
    restart: always
    ports:
      - 8001:8001
    volumes:
      - redis_insight_volume_data:/db
volumes:
  redis_volume_data:
  redis_insight_volume_data:
```
![Untitled 1](https://github.com/mohammad-fahs/redis-job-queue/assets/109221274/ecd9db59-c062-49d2-a25c-78855f016405)



# Redis Configuration in Java

→ Jedis is a Java client for [Redis](https://github.com/redis/redis) designed for performance and ease of use. I have created a maven spring boot project and added the Jedis Dependency to the POM File 

```yaml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>5.0.0</version>
</dependency>
```

⇒ To connect to Redis instance I have created a Jedis Client class 

The **`JedisClient`** class acts as a wrapper for the Jedis client, providing methods to store and retrieve string values in Redis.

```java
import org.springframework.stereotype.Component;
import redis.clients.jedis.Jedis;

@Component
public class JedisClient {
    Jedis client; // Jedis client instance for Redis interaction

    // Private constructor to initialize Jedis client with Redis server URL
    private JedisClient() {
        client = new Jedis("http://127.0.0.1:6379/");
    }

    // Method to store a string value in Redis
    public void putString(String key, String value) {
        client.set(key, value);
    }

    // Method to retrieve a string value from Redis
    public String getString(String key) {
        return client.get(key);
    }
}

```

# Job Queue Demo

In this project, I've adopted a producer-consumer pattern using Redis to implement a job queue system. The producer project is responsible for adding jobs to a Redis list, while the consumer project is tasked with processing these jobs.

**Redis as the Job Queue:**

- Redis acts as the underlying job queue, storing jobs in a first-in, first-out (FIFO) manner.
- Jobs added by the producer are stored in the Redis list, ensuring that they are processed in the order they were added.

This architecture decouples the job creation (producer) from the job processing (consumer), enabling asynchronous task handling and scalability. It also leverages Redis's efficient data structures and commands for efficient job queue management.

The key components of this implementation are as follows:

1. **Job Class:**
    - A common class shared by both the producer and consumer projects.
    - It encapsulates job-related data, simplifying the storage of string-based job information in Redis.
    
    ```java
    @Getter
    @Setter
    public class Job {
        private String data;
        public Job(String data) {
            this.data = data;
        }
    }
    ```
    
2. **Producer Project:**
    - The producer project is responsible for creating and adding jobs to the Redis list.
    - It utilizes the Job class to create job instances and adds them to the Redis list using Redis commands like **`lpush`**.
    
    ```java
    import lombok.RequiredArgsConstructor;
    import org.springframework.stereotype.Component;
    
    @Component
    @RequiredArgsConstructor
    public class JobProducer {
        private final JedisClient redis;
        public void addJob(String jobData) {
            redis.client.lpush("job_queue", jobData);
        }
    }
    
    ```
    
    ⇒ To Test the application I will use Command Line runner 
    
    ```java
    package com.bigdata.producer;
    
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.boot.CommandLineRunner;
    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    
    import java.util.Scanner;
    
    @SpringBootApplication
    public class ProducerApplication implements CommandLineRunner {
    	@Autowired
    	private JobProducer producer; // Autowire the JobProducer bean
    	
    	Scanner scanner = new Scanner(System.in); // Initialize a Scanner object to read user input
    
    	public static void main(String[] args) {
    		SpringApplication.run(ProducerApplication.class, args); // Run the Spring Boot application
    	}
    
    	@Override
    	public void run(String... args) throws Exception {
    		System.out.print("Enter your input (type 'exit' to quit): "); // Prompt the user for input
    		while (true) {
    			String input = scanner.nextLine(); // Read the user's input
    			
    			producer.addJob(input); // Add the input as a job to the producer
    			
    			if (input.equalsIgnoreCase("exit")) { // Check if the user wants to exit
    				System.out.println("Exiting..."); // Print a message indicating the program is exiting
    				break; // Exit the while loop
    			}
    		}
    
    		scanner.close(); // Close the scanner to free up resources
    	}
    }
    
    ```
    
3. **Consumer Project:**
    - The consumer project is responsible for processing jobs retrieved from the Redis list.
    - It connects to the Redis server, fetches jobs from the list (using commands like **`lpop`** or **`brpop`** for blocking retrieval), and processes each job accordingly.
    
    ```java
    import lombok.RequiredArgsConstructor;
    import org.springframework.stereotype.Component;
    
    import java.util.List;
    
    @Component
    @RequiredArgsConstructor
    public class JobConsumer {
        private final JedisClient redis;
        public void processJobs() {
            int i=0;
            while (true) {
                List<String> jobList = redis.client.brpop(0, "job_queue");
                if (jobList != null) {
                    String jobData = jobList.get(1);
                    if(jobData.equals("exit"))
                        break;
                    System.out.println("Processing job number"+ i++ +" : " + jobData);
                }
            }
        }
    }
    ```
    
    ⇒ To test the application i will use the Command Line runner
    
    ```java
    package com.bigdata.consumer;
    
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.boot.CommandLineRunner;
    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    
    @SpringBootApplication
    public class ConsumerApplication implements CommandLineRunner {
    
    	@Autowired
    	private JobConsumer consumer;
    
    	public static void main(String[] args) {
    		SpringApplication.run(ConsumerApplication.class, args);
    	}
    
    	@Override
    	public void run(String... args) throws Exception {
    		consumer.processJobs();
    	}
    }
    
    ```
    ![Untitled 2](https://github.com/mohammad-fahs/redis-job-queue/assets/109221274/a91e7720-4584-4722-8670-8d7fd097ab0b)


    ![Untitled 3](https://github.com/mohammad-fahs/redis-job-queue/assets/109221274/b783105d-4920-4a04-aac0-be4cc13f8b10)


## What If i have multiple Consumers ?

### Producer
![Untitled 4](https://github.com/mohammad-fahs/redis-job-queue/assets/109221274/bff55ec9-ba18-4682-a6af-4de65436dd2a)


### Consumer 1
![Untitled 5](https://github.com/mohammad-fahs/redis-job-queue/assets/109221274/a6b0107d-26c8-480b-aaf3-0c14aa12c40e)


### Consumer 2
![Untitled 6](https://github.com/mohammad-fahs/redis-job-queue/assets/109221274/6c410576-b740-4500-97a2-d2dae8a00539)

