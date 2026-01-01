# Secure Notes App

A serverless secure notes API built on AWS, featuring Cognito authentication, API Gateway with JWT authorization, Lambda handlers, and DynamoDB storage.

## Problem Statement

Users need a secure, scalable way to store and manage personal notes with proper authentication and authorization. This project implements a RESTful API that ensures each user can only access their own notes while leveraging AWS managed services for reliability and cost efficiency.

## Architecture

```
┌─────────────────┐
│     Client      │
│ (Postman/CLI)   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Cognito User   │
│     Pool        │
│  (Auth + JWT)   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   API Gateway   │
│ (JWT Authorizer)│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│     Lambda      │
│ (Notes Handler) │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│    DynamoDB     │
│ (SecureNotes)   │
└─────────────────┘
```

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/notes` | Create a new note |
| GET | `/notes` | List current user's notes |
| GET | `/notes/{id}` | Get a specific note |
| PUT | `/notes/{id}` | Update a note |
| DELETE | `/notes/{id}` | Delete a note |

## Technology Stack

- **Authentication**: AWS Cognito User Pool
- **API Layer**: AWS API Gateway with Cognito Authorizer
- **Compute**: AWS Lambda (Python 3.11)
- **Database**: AWS DynamoDB
- **Infrastructure**: AWS SAM / CloudFormation
- **Monitoring**: CloudWatch Alarms and Logs

## Quick Start

### Prerequisites

- AWS Account (Free Tier eligible)
- AWS CLI configured
- Python 3.11+
- SAM CLI

### Deployment

```bash
# Clone the repository
git clone https://github.com/Sakeeb91/secure-notes-app.git
cd secure-notes-app

# Deploy to AWS
sam build
sam deploy --guided
```

### Testing the API

```bash
# Sign up a new user
aws cognito-idp sign-up \
  --client-id <APP_CLIENT_ID> \
  --username user@example.com \
  --password SecurePass123!

# Get JWT token after confirmation
TOKEN=$(aws cognito-idp initiate-auth \
  --client-id <APP_CLIENT_ID> \
  --auth-flow USER_PASSWORD_AUTH \
  --auth-parameters USERNAME=user@example.com,PASSWORD=SecurePass123! \
  --query 'AuthenticationResult.IdToken' --output text)

# Create a note
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"text":"First secure note"}' \
  https://<api-id>.execute-api.<region>.amazonaws.com/prod/notes

# List notes
curl -H "Authorization: Bearer $TOKEN" \
  https://<api-id>.execute-api.<region>.amazonaws.com/prod/notes
```

## Features

- **Secure Authentication**: Cognito-managed user registration and login
- **JWT Authorization**: Stateless token validation at API Gateway
- **User Isolation**: Each user can only access their own notes
- **Structured Logging**: JSON logs for debugging and monitoring
- **CloudWatch Alerts**: Automated notifications for errors and throttling

## Future Enhancements

- [ ] Idempotency support via `Idempotency-Key` header
- [ ] Pagination with `LastEvaluatedKey`
- [ ] Search functionality via DynamoDB GSI
- [ ] File attachments via S3 presigned URLs
- [ ] OpenSearch integration for full-text search

## Cleanup

To avoid ongoing AWS charges:

```bash
# Delete the CloudFormation stack
sam delete

# Or manually delete:
# 1. API Gateway - delete API (removes stage)
# 2. Lambda - delete SecureNotesHandler
# 3. DynamoDB - delete SecureNotes table
# 4. Cognito - delete user pool
# 5. IAM - remove inline policies/roles if not reused
```

## License

MIT License

## Author

Sakeeb Rahman - [GitHub](https://github.com/Sakeeb91)
