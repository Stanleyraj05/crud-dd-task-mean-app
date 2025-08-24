Followed these steps to set up and run the project locally.

### ### 1. Clone the Repository

First, clone the project from Git repository to my local machine.

```bash
git clone https://github.com/Stanleyraj05/crud-dd-task-mean-app
cd crud-dd-task-mean-app
```

### ### 2. Set Up and Run the Backend

The backend server connects to the MongoDB database and exposes the API.

1.  **Navigate to the backend directory:**
    ```bash
    cd backend
    ```

2.  **Install dependencies:**
    ```bash
    npm install
    ```
    

3.  **Start your MongoDB database server.** If you installed it with Homebrew on macOS, run:
    ```bash
    brew services start mongodb-community
    ```
    *(For other operating systems, please follow the official MongoDB documentation to start the service.)*

4.  **Start the backend server:**
    ```bash
    node server.js
    ```
    The API server will now be running and listening on **`http://localhost:8080`**.
    <img width="1440" height="900" alt="Screenshot 2025-08-24 at 17 45 51" src="https://github.com/user-attachments/assets/64d7b86f-87ee-4912-aafa-5228f03322bb" />
    
### ### 3. Set Up and Run the Frontend

The frontend is an Angular application that communicates with the backend API.

1.  **Navigate to the frontend directory:**
    ```bash
    cd frontend
    ```

2.  **Check Node.js version.** The Angular CLI requires a modern version of Node.
    ```bash
    node -v

    nvm use 20
    ```

3.  **Install dependencies:**
    ```bash
    npm install
    ```

4.  **Install the Angular CLI.
    ```bash
    npm install -g @angular/cli
    ```

5.  **Start the frontend development server:**
    ```bash
    ng serve --port 8081
    ```
    The Angular application will now compile and run, available at **`http://localhost:8081`**.
<img width="1440" height="900" alt="Screenshot 2025-08-24 at 17 52 20" src="https://github.com/user-attachments/assets/b74afcb3-7b94-4da9-976c-7d02b9f4647b" />

---

### Step 1: Launch and Connect to AWS EC2 Instance

1.  **Launch an EC2 Instance:**
    * Navigate to the AWS EC2 console and launch a new instance.
    * Select **Ubuntu 22.04 LTS** Amazon Machine Image (AMI).
    * Configure a security group to allow inbound traffic on the following ports:
        * **SSH (Port 22):** From IP address for secure access.
        * **HTTP (Port 8088):** From anywhere (`0.0.0.0/0`) to access the application.
    * Launch the instance SSH key pair.
<img width="1440" height="900" alt="Screenshot 2025-08-24 at 20 34 11" src="https://github.com/user-attachments/assets/f4e3c6a5-6113-446e-8925-7ed1ec14772c" />

2.  **Connect via SSH:**

    ```bash
    ssh -i crud-dd-task-mean-app.pem ubuntu@44.211.126.143
    ```

### Step 2: Install and Configure Server Dependencies

1.  **Update Package Lists:**
    ```bash
    sudo apt-get update
    ```

2.  **Install Docker Engine:**
    ```bash
    sudo apt-get install -y docker.io
    sudo systemctl start docker
    sudo systemctl enable docker
    ```
   
   <img width="1440" height="900" alt="Screenshot 2025-08-24 at 20 35 13" src="https://github.com/user-attachments/assets/307b381d-8b10-44f3-a2cf-e233e015c2b1" />

<img width="1440" height="900" alt="Screenshot 2025-08-24 at 19 55 25" src="https://github.com/user-attachments/assets/3f02d761-b365-46e7-ac5a-92da3c2ca01f" />
<img width="1440" height="900" alt="Screenshot 2025-08-24 at 19 55 08" src="https://github.com/user-attachments/assets/f0a3a5c4-c5db-4df6-8522-1b34891b5df3" />


3.  **Add User to Docker Group:**
    ```bash
    sudo usermod -aG docker ${USER}
    ```

4.  **Install Docker Compose:**
    ```bash
    sudo apt-get install -y docker-compose
    ```

5.  **Install Nginx:**
    ```bash
    sudo apt-get install -y nginx
    sudo systemctl start nginx
    ```

### Step 3: Configure Nginx as a Reverse Proxy

1.  **Edit the Nginx Configuration File:**
    ```bash
    sudo nano /etc/nginx/sites-available/default
    ```

2.  **Add Server Block Configuration:**
    ```nginx
    server {
    listen 8088;
    server_name 44.211.126.143;

    # Route root traffic to the frontend container
    location / {
        proxy_pass http://localhost:8081;
        try_files $uri $uri/ /index.html;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Route API calls to the backend container
    location /api/ {
        proxy_pass http://localhost:3001/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

    ```

3.  **Test and Restart Nginx:**
    ```bash
    sudo nginx -t
    sudo systemctl restart nginx
    ```

### Step 4: Automate Deployment with GitHub Actions

```yaml
name: CI/CD Pipeline for MEAN App

on:
  push:
    branches: main

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2

    - name: Login to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}

    - name: Build and push backend image
      uses: docker/build-push-action@v4
      with:
        context: ./backend
        file: ./backend/Dockerfile
        push: true
        tags: ${{ secrets.DOCKERHUB_USERNAME }}/mean-app-backend:latest

    - name: Build and push frontend image
      uses: docker/build-push-action@v4
      with:
        context: ./frontend
        file: ./frontend/Dockerfile
        push: true
        tags: ${{ secrets.DOCKERHUB_USERNAME }}/mean-app-frontend:latest

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Deploy to VM
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.SSH_HOST }}
        username: ${{ secrets.SSH_USER }}
        key: ${{ secrets.SSH_PRIVATE_KEY }}
        script: |
          # Navigate to the project directory (clone it if it doesn't exist)
          if [ ! -d "mean-app-deployment" ]; then
            git clone https://github.com/${{ github.repository }}.git mean-app-deployment
          fi
          cd mean-app-deployment

          # Pull the latest changes from the repo
          git pull

          # Pull the latest Docker images
          docker-compose pull

          # Stop and remove old containers, then start new ones
          docker-compose down
          docker-compose up -d
```

## ✅ Accessing Application

http://44.211.126.143:8088
<img width="1440" height="900" alt="Screenshot 2025-08-24 at 20 39 09" src="https://github.com/user-attachments/assets/0fd1b15c-d0a7-49bb-a2e7-e3df82c380e7" />
