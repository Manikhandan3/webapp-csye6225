# WebApp: REST API with AWS CI/CD

This repository contains a application designed for cloud-native deployment on AWS. It provides a RESTful API for file management using AWS S3 for storage and MySQL for metadata. The project is equipped with a full CI/CD pipeline using GitHub Actions and Packer for automated builds, testing, and blue-green deployments via AWS Auto Scaling Groups.

## ✨ Features

- **Health Check Endpoint**: A `/healthz` endpoint to verify application and database connectivity
- **File Management API**: Endpoints to upload, retrieve, and delete files
- **Cloud Integration**: Uses AWS S3 for object storage and RDS (MySQL) for data persistence
- **Automated CI/CD**: GitHub Actions pipeline for testing, building AMIs with Packer, and deploying to an Auto Scaling Group
- **Infrastructure as Code**: Packer is used to codify the creation of Amazon Machine Images (AMIs)
- **Observability**:
  - **Structured Logging**: Centralized logging to AWS CloudWatch using Winston
  - **Custom Metrics**: Application metrics (API latency, database query times) sent to AWS CloudWatch via StatsD
- **Secure and Scalable**: Designed to run behind a load balancer in an Auto Scaling Group for high availability and scalability

## 🏛️ Architecture Overview

The application is deployed on EC2 instances within an Auto Scaling Group (ASG). An Application Load Balancer (ALB) routes traffic to the instances.

### Code & CI/CD
Developers push code to GitHub. A Pull Request to `main` triggers validation and testing workflows.

### Merge & Deploy
Merging a PR to `main` triggers the deployment pipeline:

1. **Test**: Runs integration tests
2. **Build**: Packer builds a new Amazon Machine Image (AMI) with the latest application code, runtime, and the CloudWatch agent
3. **Update**: A new version of the EC2 Launch Template is created with the new AMI ID
4. **Deploy**: The Auto Scaling Group performs an instance refresh, replacing old instances with new ones launched from the updated template, ensuring a zero-downtime deployment

### Application Runtime

- The application runs as a systemd service on each EC2 instance
- It connects to an external AWS RDS MySQL database for metadata
- It uses an S3 bucket for file storage
- The CloudWatch agent collects logs and metrics from each instance and sends them to AWS CloudWatch

## 🚀 CI/CD Pipeline

The CI/CD process is managed by three GitHub Actions workflows:

### tests.yml
- **Trigger**: On pull requests to `main`
- **Action**: Runs the Jest integration test suite. It spins up a temporary MySQL database in a service container to validate database interactions

### packer-validate.yml
- **Trigger**: On pull requests to `main`
- **Action**: Validates the Packer template by running `packer fmt -check` and `packer validate` to ensure the configuration is correct before merging

### packer-build.yml
- **Trigger**: On pushes (merges) to the `main` branch
- **Jobs**:
  - **integration-tests**: A final run of the tests to ensure the merged code is stable
  - **build-ami**: Uses Packer to build a new AMI. It installs runtime, the AWS CLI, the CloudWatch Agent, and the application code
  - **update-launch-template**: Creates a new version of the production AWS Launch Template, pointing to the new AMI from the previous job
  - **refresh-asg**: Triggers an instance refresh on the production Auto Scaling Group to roll out the new version

## 📖 API Specification

### Health Check

#### GET /healthz

Checks the application's health and database connectivity.

- **Request**: No body, query parameters, or URL parameters are allowed
- **Responses**:
  - `200 OK`: Service is healthy
  - `400 Bad Request`: If the request contains a body or parameters
  - `405 Method Not Allowed`: For any method other than GET
  - `503 Service Unavailable`: If the database connection fails

### File Management

All file endpoints are prefixed with `/v1`.

#### POST /v1/file

Uploads a file.

- **Request**: `multipart/form-data` with a single file field named `file`
- **Response 201 Created**:
  ```json
  {
      "id": "a9e1e2ef-c1d2-4e3f-8a4b-5d6c7b8a9e0f",
      "file_name": "example.png",
      "url": "your-s3-bucket/a9e1e2ef-c1d2-4e3f-8a4b-5d6c7b8a9e0f/a9e1e2ef-c1d2-4e3f-8a4b-5d6c7b8a9e0f-example.png",
      "upload_date": "2023-10-27"
  }
  ```
- **Other Responses**:
  - `400 Bad Request`: No file provided, invalid content type, or query parameters used
  - `401 Unauthorized`: AWS credentials error
  - `405 Method Not Allowed`: For methods other than POST
  - `503 Service Unavailable`: S3 or database error

#### GET /v1/file/:id

Retrieves metadata for a specific file.

- **Response 200 OK**:
  ```json
  {
      "id": "a9e1e2ef-c1d2-4e3f-8a4b-5d6c7b8a9e0f",
      "file_name": "example.png",
      "url": "your-s3-bucket/a9e1e2ef-c1d2-4e3f-8a4b-5d6c7b8a9e0f/a9e1e2ef-c1d2-4e3f-8a4b-5d6c7b8a9e0f-example.png",
      "upload_date": "2023-10-27T00:00:00.000Z"
  }
  ```
- **Other Responses**:
  - `404 Not Found`: If the file with the given ID does not exist
  - `405 Method Not Allowed`: For methods other than GET or DELETE
  - `503 Service Unavailable`: Database error

#### DELETE /v1/file/:id

Deletes a file from S3 and its metadata from the database.

- **Response**: `204 No Content` on successful deletion
- **Other Responses**:
  - `404 Not Found`: If the file with the given ID does not exist
  - `401 Unauthorized`: AWS credentials error
  - `405 Method Not Allowed`: For methods other than GET or DELETE
  - `503 Service Unavailable`: S3 or database error

## ⚙️ Local Development Setup

### Prerequisites

- Node.js (v20.x or higher)
- npm
- A running MySQL instance (or Docker)

### Installation

1. **Clone the repository**:
   ```bash
   git clone https://github.com/Manikhandan-CSYE6225/webapp.git
   cd webapp
   ```

2. **Install dependencies**:
   ```bash
   npm install
   ```

3. **Set up environment variables**:
   Create a `.env` file in the root of the project and add the following variables:

   ```env
   # Application Port
   PORT=3000

   # Log Level (debug, info, warn, error)
   LOG_LEVEL=info

   # Development Database Configuration
   DB_USERNAME_DEV=your_db_user
   DB_PASSWORD_DEV=your_db_password
   DB_NAME_DEV=your_db_name
   DB_HOST_DEV=localhost
   DB_DIALECT_DEV=mysql

   # Test Database Configuration (for running tests)
   DB_USERNAME_TEST=your_test_db_user
   DB_PASSWORD_TEST=your_test_db_password
   DB_NAME_TEST=your_test_db_name
   DB_HOST_TEST=localhost
   DB_DIALECT_TEST=mysql

   # AWS Configuration
   AWS_REGION=us-east-1
   S3_BUCKET=your-s3-bucket-name
   ```

   **Note**: For file uploads, you will also need to have your AWS credentials configured (e.g., via `~/.aws/credentials`).

4. **Run the application**:
   ```bash
   node index.js
   ```

   The server will start on the port specified in your `.env` file (default: 3000).

5. **Run tests**:
   Ensure your test database is running and the `_TEST` environment variables are correctly set.
   ```bash
   npm test
   ```

## ☁️ Deployment

Deployment is handled automatically by the CI/CD pipeline upon merging to the `main` branch.

### Required AWS Resources

The following resources must be provisioned in your AWS account beforehand:

- An S3 bucket for file storage
- An RDS MySQL instance
- An IAM role for the EC2 instances with permissions for S3, CloudWatch, and RDS
- An EC2 Launch Template
- An Auto Scaling Group using the Launch Template
- An Application Load Balancer pointing to the Auto Scaling Group

### Required GitHub Secrets

The GitHub Actions workflows require the following secrets to be configured in your repository settings:

- `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`: For building the AMI
- `AWS_PROD_ACCESS_KEY_ID`, `AWS_PROD_SECRET_ACCESS_KEY`: For updating the launch template and ASG
- `AWS_REGION`: The AWS region for all resources
- `DB_..._TEST`: Credentials for the test database used in CI
- `AMI_USERS`: AWS Account ID to share the created AMI with
- `LAUNCH_TEMPLATE_NAME`: The name of the production EC2 Launch Template
- `ASG_NAME`: The name of the production Auto Scaling Group
