# Real-World Tech Stack Examples in Docker Compose

Docker Compose is versatile enough to handle almost any combination of technologies. Below are detailed examples of three extremely common real-world application stacks and how they are orchestrated using `docker-compose.yml`.

---

## 1. WordPress + MySQL
*(The Classic CMS Architecture)*

WordPress powers over 40% of the web. It requires a PHP web server and a MySQL (or MariaDB) database. Using Docker Compose for WordPress is highly popular because it eliminates the complex server configuration usually required.

### Key Concepts
*   **Pre-built Images:** Both WordPress and MySQL have official, heavily optimized Docker images. We use the `image` field for both; no custom `Dockerfile` is needed.
*   **Data Persistence:** Both services *must* use volumes. MySQL needs a volume to save the actual blog posts, and WordPress needs a volume to save uploaded images, themes, and plugins.
*   **Environment Variables:** WordPress uses variables like `WORDPRESS_DB_HOST` to know exactly which container to connect to.

### Example `docker-compose.yml`
```yaml
version: '3.8'

services:
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root_password
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wp_user
      MYSQL_PASSWORD: wp_password
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - wp_net

  wordpress:
    image: wordpress:latest
    ports:
      - "8000:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wp_user
      WORDPRESS_DB_PASSWORD: wp_password
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wp_data:/var/www/html
    depends_on:
      - db
    networks:
      - wp_net

networks:
  wp_net:

volumes:
  db_data:
  wp_data:
```

---

## 2. Node.js + MongoDB
*(The Modern JavaScript Stack)*

This is the backend for the "M" and "N" in standard MEAN/MERN stacks. It pairs an asynchronous JavaScript API with a flexible NoSQL database.

### Key Concepts
*   **Hybrid Sourcing:** The database (MongoDB) uses a pre-built official `image`, while the Node.js API uses `build` to compile your custom application code from a local directory.
*   **Connection Strings:** MongoDB connections are usually formatted as a URI. The Node.js application will read the `MONGO_URI` environment variable, connecting to `mongodb://mongo_db:27017/myapp`.

### Example `docker-compose.yml`
```yaml
version: '3.8'

services:
  mongo_db:
    image: mongo:6.0
    volumes:
      - mongo_data:/data/db
    networks:
      - node_network

  node_api:
    build: ./api-source-code   # Points to a directory with a Dockerfile
    ports:
      - "3000:3000"
    environment:
      - MONGO_URI=mongodb://mongo_db:27017/myapp
      - NODE_ENV=development
    volumes:
      # Maps local code to container for live-reloading during dev
      - ./api-source-code:/usr/src/app 
      - /usr/src/app/node_modules
    depends_on:
      - mongo_db
    networks:
      - node_network

networks:
  node_network:

volumes:
  mongo_data:
```

---

## 3. Java Spring Boot + PostgreSQL
*(The Enterprise Backend Architecture)*

Java Spring Boot and PostgreSQL represent a highly robust, strictly typed, relational architecture favored by large enterprises. 

### Key Concepts
*   **Strict Dependencies:** Java applications often crash entirely if the database is not ready to accept connections when the application boots. Simply waiting for the Postgres container to *start* isn't enough; we must wait for it to be *healthy*.
*   **Overriding Application Properties:** Spring Boot conventionally uses an `application.properties` or `application.yml` file. Docker Compose allows you to override these properties dynamically using uppercase environment variables (e.g., `SPRING_DATASOURCE_URL`).

### Example `docker-compose.yml`
```yaml
version: '3.8'

services:
  postgres_db:
    image: postgres:15
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secure123
      POSTGRES_DB: enterprise_db
    volumes:
      - pg_data:/var/lib/postgresql/data
    networks:
      - spring_network
    # Essential for Java to prevent boot failures
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d enterprise_db"]
      interval: 10s
      timeout: 5s
      retries: 5

  spring_backend:
    build: ./spring-app
    ports:
      - "8080:8080"
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres_db:5432/enterprise_db
      - SPRING_DATASOURCE_USERNAME=admin
      - SPRING_DATASOURCE_PASSWORD=secure123
      - SPRING_JPA_HIBERNATE_DDL_AUTO=update
    depends_on:
      postgres_db:
        condition: service_healthy # Waits for the healthcheck to pass
    networks:
      - spring_network

networks:
  spring_network:

volumes:
  pg_data:
```
