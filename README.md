# StreamingApp

Stream premium video content, host live watch parties, and manage your catalogue with a modern microservice architecture. The platform now ships with a production-ready admin portal, real-time chat, S3-backed adaptive streaming, and a redesigned cinematic frontend experience.

## Architecture

| Service | Port | Description |
| --- | --- | --- |
| `authService` | 3001 | User authentication, registration, JWT issuance |
| `streamingService` | 3002 | Video catalogue, S3 playback endpoints, public APIs |
| `adminService` | 3003 | Dedicated admin microservice for asset management and uploads |
| `chatService` | 3004 | Websocket + REST chat for live watch parties |
| `frontend` | 3000 | React SPA with revamped UI and integrated chat |
| `mongo` | 27017 | Shared MongoDB instance |

All backend services share common database models and utilities through `backend/common`.

## Environment Configuration

Create an `.env` for each service (or export variables before running). All services accept the standard AWS credentials for S3 access.

### Auth Service (`backend/authService/.env`)
```ini
PORT=3001
MONGO_URI=mongodb://localhost:27017/streamingapp
JWT_SECRET=changeme
CLIENT_URLS=http://localhost:3000
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=ap-south-1
AWS_S3_BUCKET=
```

### Streaming Service (`backend/streamingService/.env`)
```ini
PORT=3002
MONGO_URI=mongodb://localhost:27017/streamingapp
JWT_SECRET=changeme
CLIENT_URLS=http://localhost:3000
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=ap-south-1
AWS_S3_BUCKET=
AWS_CDN_URL=
STREAMING_PUBLIC_URL=http://localhost:3002
```

### Admin Service (`backend/adminService/.env`)
```ini
PORT=3003
MONGO_URI=mongodb://localhost:27017/streamingapp
JWT_SECRET=changeme
CLIENT_URLS=http://localhost:3000
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=ap-south-1
AWS_S3_BUCKET=
```

### Chat Service (`backend/chatService/.env`)
```ini
PORT=3004
MONGO_URI=mongodb://localhost:27017/streamingapp
JWT_SECRET=changeme
CLIENT_URLS=http://localhost:3000
```

### Frontend build variables (`frontend/.env` or Docker build args)
```ini
REACT_APP_AUTH_API_URL=http://localhost:3001/api
REACT_APP_STREAMING_API_URL=http://localhost:3002/api
REACT_APP_STREAMING_PUBLIC_URL=http://localhost:3002
REACT_APP_ADMIN_API_URL=http://localhost:3003/api/admin
REACT_APP_CHAT_API_URL=http://localhost:3004/api/chat
REACT_APP_CHAT_SOCKET_URL=http://localhost:3004
```

## Running with Docker Compose

1. Populate the environment variables above (or rely on the defaults baked into `docker-compose.yml`).
2. Build and start the stack:
   ```bash
   docker-compose up --build
   ```
3. Navigate to `http://localhost:3000` for the web app.

The compose file provisions MongoDB plus all four Node.js microservices. S3 credentials are optional for local testing—you can still browse seeded metadata, but streaming requires valid S3 objects.

## Local Development

Install dependencies for each service:

```bash
# auth service
cd backend/authService && npm install

# streaming service
cd ../streamingService && npm install

# admin service
cd ../adminService && npm install

# chat service
cd ../chatService && npm install

# frontend
cd ../../frontend && npm install
```

Run the services (in separate terminals) after starting MongoDB:

```bash
cd backend/authService && npm run dev
cd backend/streamingService && npm run dev
cd backend/adminService && npm run dev
cd backend/chatService && npm run dev
cd frontend && npm start
```

## Feature Highlights

- **S3-backed adaptive streaming** with secure signed uploads for admins.
- **Dedicated admin microservice** for video ingestion, metadata management, and featured curation.
- **Real-time chat** overlay in the player (Socket.IO + persistent message history).
- **Modern React experience** featuring cinematic hero sections, dynamic carousels, and responsive design.
- **Role-aware access control** across frontend routes and backend microservices.

## Testing

Automated tests are not yet included. Recommended smoke checks:

1. Register and log in through the web UI.
2. Upload a small video + thumbnail via the admin dashboard (requires valid S3 credentials).
3. Confirm playback from the browse page and verify that chat messages broadcast between multiple browser tabs.

## License

MIT © StreamFlix Team

---

## AWS EKS Deployment — DevOps Project

The application has been containerized and deployed to Amazon EKS using Jenkins, Docker, Amazon ECR, Kubernetes, and Helm.

**Final deployed Docker image version:** `1.0.3`

**Verified Jenkins build:** `#8 — SUCCESS`

**Kubernetes deployment:** Six Running pods across two Ready worker nodes.

### Detailed Project Documentation

- [Complete AWS Deployment Guide](docs/AWS-DEPLOYMENT-GUIDE.md) — architecture, infrastructure, Docker, Jenkins CI/CD, ECR, Kubernetes, Helm, configuration, verification, and operational limitations.
- [Troubleshooting and Debugging Guide](docs/TROUBLESHOOTING.md) — deployment failures, investigation commands, root causes, fixes, and verification.
- [Screenshot Evidence Checklist](docs/screenshots/README.md) — required screenshots and naming conventions.

### Project Links

- [GitHub Repository](https://github.com/srinivassriaddepalli-creator/StreamingApp)
- [Jenkins CI/CD Pipeline](https://jenkinsacademics.herovired.com/job/StreamingApp-Srinivas-CI-CD/)
- [Live StreamFlix Application](http://a84ad60694e6040c189032ed80506985-4211009.ap-south-1.elb.amazonaws.com/)

**Note:** The current Jenkins pipeline automates image builds and ECR publishing. Helm deployment to EKS is performed manually. Amazon CloudWatch monitoring and centralized logging are configured. The frontend has two running replicas; load-based autoscaling has not been verified.

## Final Deployment and CI Verification — October 9, 2026

### Automated Jenkins CI Pipeline

The Jenkins pipeline is configured with **Poll SCM** using the schedule `H/2 * * * *`.

- **Successful build:** #9
- **Trigger:** Started by an SCM change
- **Git commit:** `42c764c`
- **Build result:** SUCCESS
- **Docker images:** Five application images built and pushed to Amazon ECR
- **Image tag:** `1.0.3`

Jenkins automatically detects GitHub changes, builds the application images, and publishes them to ECR. Deployment to Amazon EKS is performed separately using Helm.

### Amazon EKS Deployment

- **AWS region:** `ap-south-1`
- **EKS cluster:** `streamingapp-eks`
- **Kubernetes namespace:** `streamingapp`
- **Helm release:** `streamingapp`
- **Verified Helm revision:** 11
- **Application deployments:** auth, stream, admin, chat, frontend
- **Frontend replicas:** 2

### Persistent MongoDB Storage

MongoDB runs as a Kubernetes StatefulSet named `streaming-mongodb-persistent`.

- **StatefulSet:** 1/1 Ready
- **StorageClass:** `streamingapp-gp3`
- **PersistentVolumeClaim:** `mongo-data-streaming-mongodb-persistent-0`
- **EBS volume size:** 10 GiB
- **PVC status:** Bound
- **Restored database records:** 8 videos and 1 user

The application uses the persistent MongoDB service instead of the previous ephemeral MongoDB Deployment.

### Application Health Verification

Both the frontend and streaming API returned HTTP 200 after the MongoDB migration and Helm upgrade.

### Live Application

http://a84ad60694e6040c189032ed80506985-4211009.ap-south-1.elb.amazonaws.com/

### Jenkins Pipeline

https://jenkinsacademics.herovired.com/job/StreamingApp-Srinivas-CI-CD/

