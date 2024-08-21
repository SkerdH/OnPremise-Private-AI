# Secure On-Premise AI System with Private Data Management
![Intro](https://i.imgur.com/UQfaIpY.png)
## 📚 Table of Contents

1. **[Environment Setup](#environment-setup)**
2. **[Docker Installation](#docker-installation)**
3. **[OpenWebUI Setup](#openwebui-setup)**
4. **[User Management](#user-management)**
5. **[Web Search and Authentication Configuration](#web-search-and-authentication-configuration)**
6. **[Load Balancing Setup](#load-balancing-setup)**
7. **[Server Configuration](#server-configuration)**
8. **[Securing Access within a LAN](#securing-access-within-a-lan)**
9. **[Backup and Recovery](#backup-and-recovery)**


## Environment Setup

### Configure Virtual Machine or Dedicated Physical Machine
- **Operating System:** Install Ubuntu onto a virtual machine or on a dedicated physical machine.
- **Network Configuration:** Ensure the server has a static IP address within your local network to maintain consistent access. This can be set up during the installation or configured afterward using the network settings.

### Install Ollama on Linux
- **Ollama Installation:**
  ```bash
  curl -fsSL https://ollama.com/install.sh | sh
  ```
- **Note:** Ollama runs on port `11434`.
- **Verify Installation:** Check that Ollama is running by executing:
  ```bash
  systemctl status ollama
  ```

### Install LLaMA 3 Model
- **Pull the LLaMA 3 Model:**
   ```bash
  ollama pull llama3
  ```

## Docker Installation
```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
## Verify Docker installation
```bash 
sudo docker --version
```

## OpenWebUI Setup

### Download and Run OpenWebUI
- **Run OpenWebUI Docker Container:**
  ```
  sudo docker run -d --network=host -v open-webui:/app/backend/data \-e OLLAMA_BASE_URL=http://127.0.0.1:11434 \
  --name open-webui --restart always ghcr.io/open-webui/open-webui:main```
- **Port Information:** OpenWebUI runs on port `8080`.
- **Access Configuration:** Open your browser and go to `http://localhost:8080`.

### Configuration for Local Network Access
- **Local Network Upload:** Configure OpenWebUI to be accessible on the local network by ensuring the Docker container is running on a machine with a static IP. Use the static IP instead of `localhost` to access the interface across the network.

## User Management

### Admin Panel - Add User Feature

In the Admin Panel, administrators can manage users efficiently through a user-friendly interface. Below is a screenshot of the **Add User** dialog, where an admin can add new users to the system.

![Admin Panel - Add User](https://i.imgur.com/Kkj21yG.png)

### Features of the Add User Interface:
- **Role Assignment:** Administrators can assign roles to new users, such as "User" or "Admin." This ensures that users are granted appropriate permissions based on their responsibilities.
- **User Information:** The form requires essential details like the user’s full name, email address, and password. These fields ensure that each new user has a unique identity within the system.
- **CSV Import:** Besides manually entering user details, admins can also import multiple users at once using a CSV file, streamlining the process of adding users in bulk.

## Web Search and Authentication Configuration

### Web Search Integration
- **Enable Web Search in OpenWebUI:**
  ```
  services:
    open-webui:
      environment:
        ENABLE_RAG_WEB_SEARCH: True
        RAG_WEB_SEARCH_ENGINE: "searxng"
        RAG_WEB_SEARCH_RESULT_COUNT: 3
        RAG_WEB_SEARCH_CONCURRENT_REQUESTS: 10
        SEARXNG_QUERY_URL: "http://searxng:8080/search?q=<query>"

    searxng:
      image: searxng/searxng:latest
      container_name: searxng
      ports:
        - "8080:8080"
      volumes:
        - ./searxng:/etc/searxng
      restart: always
  ```
- **Explanation:** SearxNG is a privacy-respecting metasearch engine that aggregates results from multiple search engines. This setup integrates SearxNG with OpenWebUI for enhanced search capabilities.

### Configure OAuth2 Authentication
- **Setup OAuth2-Proxy for Authentication:**
  ```
  services:
    open-webui:
      image: ghcr.io/open-webui/open-webui:main
      volumes:
        - open-webui:/app/backend/data
      environment:
        - 'HOST=127.0.0.1'
        - 'WEBUI_AUTH_TRUSTED_EMAIL_HEADER=X-Forwarded-Email'
        - 'WEBUI_AUTH_TRUSTED_NAME_HEADER=X-Forwarded-User'
      restart: unless-stopped

    oauth2-proxy:
      image: quay.io/oauth2-proxy/oauth2-proxy:v7.6.0
      environment:
        OAUTH2_PROXY_HTTP_ADDRESS: 0.0.0.0:4180
        OAUTH2_PROXY_UPSTREAMS: http://open-webui:8080/
        OAUTH2_PROXY_PROVIDER: google
        OAUTH2_PROXY_CLIENT_ID: REPLACEME_OAUTH_CLIENT_ID
        OAUTH2_PROXY_CLIENT_SECRET: REPLACEME_OAUTH_CLIENT_ID
        OAUTH2_PROXY_EMAIL_DOMAINS: REPLACEME_ALLOWED_EMAIL_DOMAINS
        OAUTH2_PROXY_REDIRECT_URL: REPLACEME_OAUTH_CALLBACK_URL
        OAUTH2_PROXY_COOKIE_SECRET: REPLACEME_COOKIE_SECRET
        OAUTH2_PROXY_COOKIE_SECURE: "false"
      restart: unless-stopped
      ports:
        - 4180:4180/tcp
  ```
- **Explanation:** This configuration sets up OAuth2 authentication for the OpenWebUI interface. It uses an OAuth2-Proxy to handle authentication, which can be configured with providers like Google or Azure AD. Replace the placeholders (`REPLACEME_...`) with actual credentials and configuration values.

## Load Balancing Setup

### Configure Load Balancer with Docker
- **Run Docker Container with Load Balancing:**
 ```bash
  docker run -d -p 3000:8080 \
  -v open-webui:/app/backend/data \
  -e OLLAMA_BASE_URLS="http://ollama-one:11434;http://ollama-two:11434" \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```
- **Explanation:** Load balancing distributes incoming traffic across multiple instances of the `ollama` service to enhance performance and reliability. This Docker command runs the OpenWebUI with load balancing enabled for the specified instances.



## Securing Access within a LAN

To ensure secure access to the AI system within a LAN for an office environment, follow these steps:

### Network Segmentation

- **Virtual LAN (VLAN):**  
  Use VLANs to segment the network and limit access to sensitive resources. Place the AI system in a dedicated VLAN, and allow access only from specific devices or subnets. Refer to your network equipment documentation for setting up VLANs.

- **Access Control Lists (ACLs):**  
  Implement ACLs on your routers or switches to control which devices can access the AI system based on IP addresses. ACLs can typically be configured through the management interface of your network devices.

### Encryption

- **TLS/SSL:**  
  Secure all communications between users and the AI system with TLS/SSL. This can be done by setting up an NGINX or Apache reverse proxy that handles HTTPS termination and forwards requests to the OpenWebUI running on HTTP.

- **Internal Certificates:**  
  Use self-signed certificates or a local Certificate Authority (CA) to issue SSL certificates for internal services. Tools like OpenSSL or a dedicated internal CA service can be used to generate and distribute certificates.

### Firewall Configuration

- **Restrict Port Access:**  
  Only allow necessary ports (e.g., 8080 for OpenWebUI, 11434 for Ollama) through the firewall. Block all other ports by default to minimize exposure to potential threats.

- **IP Whitelisting:**  
  Limit access to the AI system only to known IP addresses within your office network. Configure this within your firewall or network equipment.

### Authentication and Authorization

- **OAuth2 Implementation:**  
  Ensure that only authenticated users can access the OpenWebUI interface. Set up OAuth2 with a robust provider (e.g., Google, Azure AD) as shown in the OAuth2 Authentication section above.

- **Role-Based Access Control (RBAC):**  
  Implement RBAC within the OpenWebUI to restrict access based on user roles. This ensures that users only have access to the data and features they need.

### Monitoring and Logging

- **Log Access:**  
  Implement logging for all access to the AI system, including who accessed it and when. Use tools like ELK Stack (Elasticsearch, Logstash, Kibana) for centralized logging.

- **Monitor Network Traffic:**  
  Use tools like Suricata, Wireshark, or a network monitoring solution to keep an eye on traffic to and from the AI system, looking for any unusual activity. Regularly review logs for potential security incidents.

### Regular Security Audits

- **Security Audits:**  
  Perform regular security audits and vulnerability scans on the AI system and its associated network to ensure there are no exploitable vulnerabilities. Tools like OpenVAS or Nessus can be used for vulnerability scanning.

## Backup and Recovery

### Backup Strategy
- **Docker Volumes:** Regularly back up Docker volumes (`open-webui:/app/backend/data`) that store important data for the OpenWebUI.
- **Configuration Files:** Ensure that configuration files (e.g., Apache, Docker, OAuth2 settings) are backed up and version-controlled.
- **Database Backups:** If the system uses a database, schedule regular backups and test recovery procedures.

### Recovery Plan
- Document the steps required to restore the AI system from backups, including redeploying Docker containers, restoring configuration files, and verifying that the system is fully operational.


