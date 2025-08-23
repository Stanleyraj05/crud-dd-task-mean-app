# frontend.Dockerfile

# Stage 1: Build the Angular application
FROM node:18-alpine AS build

WORKDIR /usr/src/app

COPY frontend/package*.json ./

RUN npm install

COPY . .

# Build the application for production
RUN npm run build

# Stage 2: Serve the application from a lightweight Nginx server
FROM nginx:1.23-alpine

# Copy the build output from the 'build' stage
COPY --from=build /usr/src/app/dist/crud-dd-task /usr/share/nginx/html

# Expose port 80
EXPOSE 80

# When the container starts, nginx will be started by default
