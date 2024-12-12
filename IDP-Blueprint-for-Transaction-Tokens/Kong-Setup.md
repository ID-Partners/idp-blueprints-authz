Certainly! Setting up Kong Gateway in Docker to test the token exchange process with PingFederate involves several steps. Below is a comprehensive guide that walks you through:
	1.	Prerequisites
	2.	Setting Up Docker Environment
	3.	Creating a Custom Kong Docker Image with Your Plugin
	4.	Configuring Docker Compose
	5.	Running Kong and Dependencies
	6.	Configuring Kong for Token Exchange
	7.	Testing the Setup
	8.	Troubleshooting Tips

1. Prerequisites

Before you begin, ensure you have the following installed on your machine:
	•	Docker: Install Docker
	•	Docker Compose: Install Docker Compose
	•	Git: Install Git
	•	Basic Knowledge of Docker and Kong: Familiarity with Docker commands and Kong’s architecture is beneficial.

2. Setting Up Docker Environment

We’ll use Docker Compose to orchestrate multiple containers, including Kong, PostgreSQL (as Kong’s database), and any mock services needed for testing.

2.1. Directory Structure

Create a project directory to hold all necessary files.

mkdir kong-pingfed-token-exchange
cd kong-pingfed-token-exchange

2.2. Clone or Create Your Custom Plugin

Assuming you’ve already developed your custom PingFed Token Exchange plugin (as described in previous steps), ensure it’s in a directory structure suitable for Docker.

If not, here’s a minimal example structure:

mkdir -p kong-plugin-pingfed-token-exchange
cd kong-plugin-pingfed-token-exchange
# Add your plugin files here (handler.lua, schema.lua, rockspec, etc.)

Ensure all necessary files (handler.lua, schema.lua, rockspec, etc.) are present.

3. Creating a Custom Kong Docker Image with Your Plugin

To use your custom plugin with Kong in Docker, you need to build a custom Docker image that includes your plugin.

3.1. Dockerfile for Custom Kong Image

Create a Dockerfile in the root of your project directory (kong-pingfed-token-exchange):

# Use the official Kong base image
FROM kong:3.3

# Set environment variables for LuaRocks
ENV KONG_PLUGINS=bundled,pingfed-token-exchange

# Install LuaRocks dependencies
RUN apk add --no-cache --update \
    git \
    unzip \
    && rm -rf /var/cache/apk/*

# Install dependencies and your custom plugin
COPY kong-plugin-pingfed-token-exchange /usr/local/share/lua/5.1/kong/plugins/pingfed-token-exchange

# Set permissions
RUN chmod -R 755 /usr/local/share/lua/5.1/kong/plugins/pingfed-token-exchange

# Install LuaRocks dependencies and your plugin
RUN luarocks install kong-plugin-pingfed-token-exchange-0.1.0-1.rockspec

# Expose Kong ports
EXPOSE 8000 8443 8001 8444

Explanation:
	•	Base Image: Starts from the official Kong Docker image.
	•	Plugins Configuration: Adds your custom plugin (pingfed-token-exchange) to the list of enabled plugins.
	•	Dependencies: Installs necessary tools (git, unzip) and copies your plugin into the image.
	•	LuaRocks Installation: Uses LuaRocks to install your plugin based on the rockspec file.
	•	Ports: Exposes Kong’s standard proxy and admin ports.

3.2. Building the Custom Kong Image

Ensure your custom plugin directory and rockspec file are correctly set up. Then, build the Docker image:

docker build -t custom-kong:latest .

Note: Building might take a few minutes as it installs dependencies.

4. Configuring Docker Compose

We’ll use Docker Compose to define and run multi-container Docker applications. Create a docker-compose.yml file in your project directory:

version: '3.8'

services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: kong
      POSTGRES_DB: kong
      POSTGRES_PASSWORD: kong
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  kong:
    image: custom-kong:latest
    restart: always
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: postgres
      KONG_PG_PASSWORD: kong
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_ADMIN_LISTEN: 0.0.0.0:8001, 0.0.0.0:8444 ssl
      KONG_PROXY_LISTEN: 0.0.0.0:8000, 0.0.0.0:8443 ssl
      KONG_PLUGINS: bundled,pingfed-token-exchange
    ports:
      - "8000:8000"  # Proxy
      - "8443:8443"  # Proxy SSL
      - "8001:8001"  # Admin
      - "8444:8444"  # Admin SSL
    depends_on:
      - postgres
    volumes:
      - ./kong-plugin-pingfed-token-exchange:/usr/local/share/lua/5.1/kong/plugins/pingfed-token-exchange

  # Optional: Mock PingFederate Server (for testing)
  mock-pingfed:
    image: kennethreitz/httpbin
    ports:
      - "8080:80"
    command: ["httpbin"]

volumes:
  postgres_data:

Explanation:
	•	postgres: Sets up a PostgreSQL database for Kong.
	•	kong: Uses the custom Kong image, links to PostgreSQL, and exposes necessary ports.
	•	mock-pingfed: An optional mock service to simulate PingFederate’s Token Exchange endpoint using httpbin. Replace or extend this with a more accurate mock if needed.
	•	Volumes: Persists PostgreSQL data and mounts the custom plugin directory into the Kong container.

Important:
	•	Ensure the mock-pingfed service accurately represents PingFederate’s Token Exchange API for testing. For more accurate testing, consider using a mock server like Mockoon or WireMock to simulate PingFed’s responses.

5. Running Kong and Dependencies

With Docker Compose configured, you can now start the services.

5.1. Start Services

Run the following command in your project directory:

docker-compose up -d

Explanation:
	•	-d: Runs containers in detached mode (in the background).

5.2. Verify Services are Running

Check the status of the containers:

docker-compose ps

You should see postgres, kong, and optionally mock-pingfed running.

5.3. Initialize Kong’s Database

Once the services are up, you need to initialize Kong’s database schema.

docker-compose exec kong kong migrations bootstrap

Note: This command should only be run once. If you’re resetting the database, use kong migrations reset.

6. Configuring Kong for Token Exchange

Now that Kong and its dependencies are running, configure Kong to handle the token exchange using your custom plugin.

6.1. Create a Mock Upstream Service

For testing, create a simple upstream service that Kong can proxy to. We’ll use httpbin as a placeholder.

# No action needed if using mock-pingfed. If you have another mock service, set it up similarly.

Alternatively, you can set up a simple HTTP service:

docker run -d --name mock-upstream -p 8081:80 kennethreitz/httpbin

6.2. Add the Upstream Service and Route in Kong

Use Kong’s Admin API to add a service and a route.

# Add a service pointing to your upstream API (e.g., mock-upstream)
curl -i -X POST http://localhost:8001/services/ \
  --data "name=mock-upstream-service" \
  --data "url=http://mock-upstream:80"

# Add a route for the service
curl -i -X POST http://localhost:8001/services/mock-upstream-service/routes \
  --data "paths[]=/exchange-token" \
  --data "methods[]=POST"

Explanation:
	•	Service: Represents the upstream API that Kong proxies to.
	•	Route: Defines how clients access the service. In this case, clients will send POST requests to /exchange-token.

6.3. Apply the Custom Plugin to the Route

Configure the pingfed-token-exchange plugin on the /exchange-token route.

# Retrieve the route ID (replace <route-name> with your actual route name if different)
ROUTE_ID=$(curl -s http://localhost:8001/routes | jq -r '.data[] | select(.paths[]? == "/exchange-token") | .id')

# Apply the plugin
curl -i -X POST http://localhost:8001/routes/${ROUTE_ID}/plugins \
  --data "name=pingfed-token-exchange" \
  --data "config.pingfed_token_endpoint=http://mock-pingfed:80/post" \
  --data "config.client_id=your_pingfed_client_id" \
  --data "config.client_secret=your_pingfed_client_secret" \
  --data "config.scope=read write" \
  --data "config.additional_params[resource]=my_resource"

Explanation:
	•	pingfed_token_endpoint: Points to the mock-pingfed service’s endpoint (http://mock-pingfed:80/post using httpbin for testing).
	•	client_id & client_secret: Use dummy values for testing purposes. Replace with real credentials when connecting to actual PingFederate.
	•	scope: Desired scopes for Token B.
	•	additional_params: Any extra parameters required by your token exchange logic.

Note:
	•	Using httpbin as Mock PingFed:
	•	The http://mock-pingfed:80/post endpoint will echo back the request in a JSON format, which can help verify that Kong is correctly forwarding the token exchange request.
	•	For more accurate testing, consider customizing the mock server to return a valid token exchange response.

7. Testing the Setup

With everything configured, you can now test the token exchange flow.

7.1. Obtain a Mock Token A

Since we’re using a mock setup, create a dummy Token A. For example:

TOKEN_A="dummy_token_a"

7.2. Make a Token Exchange Request

Send a POST request to Kong’s /exchange-token endpoint with Token A in the Authorization header.

curl -X POST http://localhost:8000/exchange-token \
  -H "Authorization: Bearer ${TOKEN_A}" \
  -H "Content-Type: application/json" \
  -d '{}'

7.3. Expected Response

Since we’re using httpbin as a mock PingFederate, the response will be an echoed version of the request sent to PingFed. A typical httpbin response for the /post endpoint looks like:

{
  "args": {},
  "data": "",
  "files": {},
  "form": {
    "grant_type": "urn:ietf:params:oauth:grant-type:token-exchange",
    "subject_token": "dummy_token_a",
    "subject_token_type": "urn:ietf:params:oauth:token-type:access_token",
    "client_id": "your_pingfed_client_id",
    "client_secret": "your_pingfed_client_secret",
    "scope": "read write",
    "resource": "my_resource"
  },
  "headers": {
    "Accept": "*/*",
    "Authorization": "Bearer dummy_token_a",
    "Content-Length": "129",
    "Content-Type": "application/x-www-form-urlencoded",
    "Host": "mock-pingfed:80",
    "User-Agent": "curl/7.68.0",
    "X-Amzn-Trace-Id": "Root=1-..."
  },
  "json": null,
  "origin": "X.X.X.X",
  "url": "http://mock-pingfed/post"
}

Note: Since httpbin is just echoing the request, it won’t return a valid Token B. To properly test token exchange, you need to simulate a successful response from PingFederate. You can achieve this by:
	•	Using a More Advanced Mock Server: Set up a mock server that returns a valid token response based on the request.
	•	Implementing a Simple Mock in mock-pingfed: Replace httpbin with a custom script that returns a predetermined JSON response.

7.4. Using a Custom Mock PingFederate Response

To simulate PingFederate returning a Token B, you can use a lightweight server like Express with Node.js or a simple Python Flask app. Below is an example using Express.

7.4.1. Create a Simple Mock PingFederate Server

Create a new directory mock-pingfed-server and add an index.js file:

mkdir mock-pingfed-server
cd mock-pingfed-server

index.js:

const express = require('express');
const bodyParser = require('body-parser');
const app = express();
const port = 80;

app.use(bodyParser.urlencoded({ extended: true }));

app.post('/as/token.exchange', (req, res) => {
  const { grant_type, subject_token, client_id, client_secret, scope, resource } = req.body;

  // Simple validation (extend as needed)
  if (grant_type !== 'urn:ietf:params:oauth:grant-type:token-exchange') {
    return res.status(400).json({ error: 'unsupported_grant_type' });
  }

  if (!subject_token || !client_id || !client_secret) {
    return res.status(400).json({ error: 'invalid_request' });
  }

  // Respond with a mock Token B
  res.json({
    access_token: 'mock_token_b',
    token_type: 'bearer',
    expires_in: 3600,
    scope: scope || 'read write',
  });
});

app.listen(port, () => {
  console.log(`Mock PingFederate Token Exchange server running on port ${port}`);
});

Explanation:
	•	Endpoint: /as/token.exchange mimics PingFederate’s Token Exchange endpoint.
	•	Validation: Checks for required parameters.
	•	Response: Returns a mock Token B (mock_token_b) with specified scopes.

7.4.2. Dockerize the Mock Server

Create a Dockerfile for the mock server.

Dockerfile:

FROM node:14-alpine

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm install

COPY index.js ./

EXPOSE 80

CMD ["node", "index.js"]

package.json:

{
  "name": "mock-pingfed",
  "version": "1.0.0",
  "description": "Mock PingFederate Token Exchange Server",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "body-parser": "^1.19.0",
    "express": "^4.17.1"
  }
}

Build and Run the Mock Server:

# From the mock-pingfed-server directory
docker build -t mock-pingfed-server:latest .

7.4.3. Update Docker Compose to Use the Custom Mock Server

Modify your docker-compose.yml to replace httpbin with your custom mock server.

version: '3.8'

services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: kong
      POSTGRES_DB: kong
      POSTGRES_PASSWORD: kong
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  kong:
    image: custom-kong:latest
    restart: always
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: postgres
      KONG_PG_PASSWORD: kong
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_ADMIN_LISTEN: 0.0.0.0:8001, 0.0.0.0:8444 ssl
      KONG_PROXY_LISTEN: 0.0.0.0:8000, 0.0.0.0:8443 ssl
      KONG_PLUGINS: bundled,pingfed-token-exchange
    ports:
      - "8000:8000"  # Proxy
      - "8443:8443"  # Proxy SSL
      - "8001:8001"  # Admin
      - "8444:8444"  # Admin SSL
    depends_on:
      - postgres
      - mock-pingfed-server
    volumes:
      - ./kong-plugin-pingfed-token-exchange:/usr/local/share/lua/5.1/kong/plugins/pingfed-token-exchange

  mock-pingfed-server:
    image: mock-pingfed-server:latest
    build:
      context: ./mock-pingfed-server
    ports:
      - "8080:80"

  # Optional: Mock Upstream Service (if not using mock-upstream container)
  mock-upstream:
    image: kennethreitz/httpbin
    ports:
      - "8081:80"

volumes:
  postgres_data:

Explanation:
	•	mock-pingfed-server: Builds and runs your custom mock PingFederate server.
	•	Remove mock-pingfed service if replacing it with mock-pingfed-server.

Rebuild and Restart Services:

docker-compose down
docker-compose up -d --build

Note: The --build flag ensures that Docker builds the updated images, including your custom mock server.

7.4.4. Update Kong Plugin Configuration

Since the mock-pingfed-server is now listening on mock-pingfed-server:80 within the Docker network, update the plugin’s pingfed_token_endpoint accordingly.

curl -i -X POST http://localhost:8001/routes/${ROUTE_ID}/plugins \
  --data "name=pingfed-token-exchange" \
  --data "config.pingfed_token_endpoint=http://mock-pingfed-server/as/token.exchange" \
  --data "config.client_id=your_pingfed_client_id" \
  --data "config.client_secret=your_pingfed_client_secret" \
  --data "config.scope=read write" \
  --data "config.additional_params[resource]=my_resource"

Note: Ensure the pingfed_token_endpoint matches the path your mock server is listening on (/as/token.exchange).

8. Testing the Setup

Now, perform end-to-end testing to ensure everything works as expected.

8.1. Obtain Token A

For testing, use a dummy Token A:

TOKEN_A="dummy_token_a"

8.2. Make a Token Exchange Request

Send a POST request to Kong’s /exchange-token endpoint with Token A.

curl -X POST http://localhost:8000/exchange-token \
  -H "Authorization: Bearer ${TOKEN_A}" \
  -H "Content-Type: application/json" \
  -d '{}'

8.3. Expected Response

The response should mimic PingFederate’s Token Exchange response as defined in your mock server.

{
  "access_token": "mock_token_b",
  "token_type": "bearer",
  "expires_in": 3600,
  "scope": "read write"
}

Explanation:
	•	access_token: The new Token B issued by PingFed (mocked as mock_token_b).
	•	token_type: Typically “bearer”.
	•	expires_in: Token validity duration in seconds.
	•	scope: Scopes assigned to Token B.

8.4. Use Token B to Access Upstream Service

Use the received Token B to make a request to a protected upstream service (if any). For example:

curl -X GET http://localhost:8000/exchange-token \
  -H "Authorization: Bearer mock_token_b" \
  -H "Content-Type: application/json" \
  -d '{}'

Note: Since exchange-token is set to proxy to mock-upstream-service, ensure your mock upstream service handles the request appropriately.

Expected Response:

Depending on your upstream service configuration, you might receive a success response or any predefined mock response.

9. Troubleshooting Tips

If you encounter issues during setup or testing, consider the following:

9.1. Check Container Logs

View logs for each service to identify errors.

# Kong Logs
docker-compose logs -f kong

# Mock PingFed Server Logs
docker-compose logs -f mock-pingfed-server

# Postgres Logs
docker-compose logs -f postgres

9.2. Verify Plugin Installation

Ensure your custom plugin is correctly installed and loaded by Kong.

# List all plugins to verify your custom plugin is present
curl -s http://localhost:8001/plugins | jq '.data[].name'

You should see "pingfed-token-exchange" in the list.

9.3. Validate Network Connectivity

Ensure that Kong can communicate with the mock-pingfed-server.

# Exec into the Kong container
docker-compose exec kong sh

# Inside Kong container, ping mock-pingfed-server
ping -c 3 mock-pingfed-server

# Or use curl to test connectivity
curl http://mock-pingfed-server/as/token.exchange

9.4. Validate Admin API Accessibility

Ensure Kong’s Admin API is accessible.

curl -i http://localhost:8001/

You should receive Kong’s Admin API information.

9.5. Test Plugin Logic

Ensure that the plugin correctly extracts the bearer token, forwards it to PingFed, and returns the expected response.
	•	Missing Authorization Header: Should return 401 Unauthorized.
	•	Invalid Authorization Header: Should return 401 Unauthorized.
	•	Valid Token Exchange: Should return Token B as per your mock server.

10. Security Considerations

While this setup is for testing purposes, it’s crucial to be aware of security best practices for production environments:
	•	Use Secure Secrets: Replace dummy client_id and client_secret with secure, randomly generated values.
	•	HTTPS: Ensure all services communicate over HTTPS to protect tokens in transit.
	•	Environment Variables: Use environment variables or Docker secrets to manage sensitive configurations.
	•	Access Controls: Restrict access to Kong’s Admin API to trusted networks or use authentication mechanisms.
	•	Rate Limiting: Implement rate limiting to prevent abuse of the token exchange endpoint.

11. Cleaning Up

After testing, you can stop and remove the containers to free up resources.

docker-compose down

To remove associated volumes (e.g., PostgreSQL data), add the -v flag:

docker-compose down -v

Summary

You’ve successfully set up Kong Gateway in Docker with a custom PingFed Token Exchange plugin. This setup allows Kong to accept a bearer token from clients, exchange it with PingFederate (mocked in this guide), and return a new token to the client. This environment is ideal for testing and development purposes.

Key Steps Covered:
	1.	Environment Setup: Prepared Docker and project structure.
	2.	Custom Plugin Integration: Built a custom Kong Docker image with the token exchange plugin.
	3.	Service Configuration: Defined services and routes in Kong via Docker Compose.
	4.	Mock Server Setup: Created a mock PingFederate server to simulate token exchange responses.
	5.	Testing: Performed end-to-end testing of the token exchange flow.
	6.	Troubleshooting: Provided tips to resolve common issues.

Next Steps:
	•	Enhance Mock Server: Make the mock PingFederate server more robust to handle various test cases.
	•	Implement Token Validation: Extend the plugin to perform more rigorous validation of Token A.
	•	Secure the Setup: Apply security best practices for production readiness.
	•	Deploy to Production: Once tested, consider deploying Kong and your plugin to a production environment, possibly using Kubernetes for scalability.

Feel free to reach out if you need further assistance or encounter specific challenges during your setup!