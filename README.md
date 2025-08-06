# springboot-hello-world

This project is a simple Spring Boot microservice designed for deployment in an AWS EKS (Elastic Kubernetes Service) environment.  
It exposes a REST API endpoint that returns a greeting message.

## Features

- REST API endpoint at `/` that returns "Hello from Spring Boot!"
- Dockerfile for containerization
- Kubernetes manifests for deployment and service configuration

## Technologies

- Java 17
- Spring Boot 3.2.x
- Docker
- Kubernetes

## Usage

1. Build the project with Maven.
2. Build and push the Docker image to your container registry.
3. Deploy to Kubernetes using the manifests in the `k8s/` directory.

## Endpoints

- `GET /` → Returns greeting message
