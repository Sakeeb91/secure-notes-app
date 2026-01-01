# Secure Notes App - Implementation Plan

## Expert Role: AWS Serverless Backend Engineer

This role is optimal because the project requires:
- Deep understanding of AWS Cognito authentication flows and JWT handling
- Experience designing DynamoDB data models with proper partition/sort key strategies
- Knowledge of API Gateway integration patterns and Cognito authorizers
- Lambda best practices for cold starts, error handling, and structured logging
- Cost optimization strategies within AWS Free Tier limits

---

## System Architecture

### High-Level Component Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                           AWS Cloud                                   │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │                        API Gateway                               │ │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐  │ │
│  │  │ /notes POST │    │ /notes GET  │    │ /notes/{id} GET/PUT │  │ │
│  │  └──────┬──────┘    └──────┬──────┘    └──────────┬──────────┘  │ │
│  │         │                  │                      │              │ │
│  │         └──────────────────┼──────────────────────┘              │ │
│  │                            │                                      │ │
│  │                    ┌───────▼───────┐                             │ │
│  │                    │   Cognito     │                             │ │
│  │                    │  Authorizer   │                             │ │
│  │                    └───────┬───────┘                             │ │
│  └────────────────────────────┼─────────────────────────────────────┘ │
│                               │                                       │
│  ┌────────────────────────────▼─────────────────────────────────────┐ │
│  │                      Lambda Function                              │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │                   SecureNotesHandler                         │ │ │
│  │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐│ │ │
│  │  │  │ create() │ │  list()  │ │  get()   │ │ update()/delete()││ │ │
│  │  │  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘│ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────┬─────────────────────────────────────┘ │
│                               │                                       │
│  ┌────────────────────────────▼─────────────────────────────────────┐ │
│  │                        DynamoDB                                   │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │                    SecureNotes Table                         │ │ │
│  │  │  PK: userId    SK: noteId    Attrs: text, createdAt, etc.   │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │                    Cognito User Pool                            │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐│   │
│  │  │ User SignUp  │  │ User SignIn  │  │ JWT Token Generation   ││   │
│  │  └──────────────┘  └──────────────┘  └────────────────────────┘│   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │                      CloudWatch                                 │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐│   │
│  │  │ Lambda Logs  │  │  API Logs    │  │  Alarms (Errors/Throt) ││   │
│  │  └──────────────┘  └──────────────┘  └────────────────────────┘│   │
│  └────────────────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────────┘

                              ▲
                              │ HTTPS
                              │
┌─────────────────────────────┴─────────────────────────────────────────┐
│                            Client                                      │
│         (curl / Postman / Web App / CLI)                              │
└───────────────────────────────────────────────────────────────────────┘
```

### Data Flow Sequence

```
1. User Registration/Login
   Client → Cognito User Pool → JWT Token returned

2. API Request with JWT
   Client (+ JWT) → API Gateway → Cognito Authorizer validates JWT
                                          ↓
                                   Lambda receives request + userId from claims
                                          ↓
                                   DynamoDB CRUD operation (scoped to userId)
                                          ↓
                                   Response returned to Client
```

---

## Technology Choices

| Component | Choice | Rationale | Tradeoffs | Fallback |
|-----------|--------|-----------|-----------|----------|
| **Auth** | Cognito User Pool | Managed auth, Free Tier (50K MAU), built-in JWT | Vendor lock-in | Auth0 Free Tier |
| **API** | API Gateway REST | Native Cognito integration, Free Tier (1M calls/mo) | Cold start latency | HTTP API (cheaper but less features) |
| **Compute** | Lambda (Python 3.11) | Familiar language, boto3 included, Free Tier (1M requests) | Cold starts | Keep functions warm with EventBridge |
| **Database** | DynamoDB | Fully managed, Free Tier (25GB), single-digit ms latency | Query flexibility | SQLite on Lambda (not recommended) |
| **IaC** | AWS SAM | Simplified Lambda/API Gateway, local testing support | Less flexible than Terraform | Plain CloudFormation |
| **Logging** | CloudWatch Logs | Integrated with Lambda, structured JSON logs | Cost at scale | Powertools for AWS Lambda |

### Key Learning Concepts

Before implementing, ensure you understand:
- **JWT Structure**: Header, Payload, Signature - and how Cognito creates them
- **DynamoDB Key Design**: Partition key (userId) + Sort key (noteId) enables efficient queries
- **IAM Least Privilege**: Lambda needs only `dynamodb:PutItem`, `GetItem`, `Query`, `DeleteItem`
- **API Gateway Integration**: Proxy vs non-proxy integration (we use proxy)

---

## Phased Implementation Plan

### Phase 1: Infrastructure Foundation

**Scope**: Set up AWS SAM project structure and deploy empty stack

**Files to Create**:
```
secure-notes-app/
├── template.yaml           # SAM template
├── src/
│   └── handlers/
│       └── __init__.py
├── tests/
│   └── __init__.py
├── samconfig.toml          # SAM deployment config
└── requirements.txt
```

**Deliverables**:
- [ ] SAM template with placeholder Lambda
- [ ] Successful `sam build` and `sam deploy`
- [ ] Stack visible in CloudFormation console

**Verification**:
```bash
sam build
sam deploy --guided
aws cloudformation describe-stacks --stack-name secure-notes-app --query 'Stacks[0].StackStatus'
# Expected: "CREATE_COMPLETE"
```

**Technical Challenges**:
- SAM CLI installation and AWS credentials configuration
- Understanding SAM template syntax (YAML anchors, intrinsic functions)

**Definition of Done**:
- `sam deploy` succeeds without errors
- Stack appears in AWS Console
- Can invoke placeholder Lambda from console

---

### Phase 2: DynamoDB Table Setup

**Scope**: Create DynamoDB table with proper key schema

**Files to Modify**:
- `template.yaml` - Add DynamoDB resource

**DynamoDB Schema**:
```
Table: SecureNotes
├── Partition Key: userId (String)
├── Sort Key: noteId (String)
├── Attributes:
│   ├── text (String)
│   ├── createdAt (String, ISO 8601)
│   └── updatedAt (String, ISO 8601)
└── Billing: PAY_PER_REQUEST (On-Demand, Free Tier eligible)
```

**Template Addition**:
```yaml
Resources:
  SecureNotesTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: SecureNotes
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: userId
          AttributeType: S
        - AttributeName: noteId
          AttributeType: S
      KeySchema:
        - AttributeName: userId
          KeyType: HASH
        - AttributeName: noteId
          KeyType: RANGE
```

**Deliverables**:
- [ ] DynamoDB table created via SAM
- [ ] Can manually insert/query items via AWS Console

**Verification**:
```bash
aws dynamodb describe-table --table-name SecureNotes --query 'Table.TableStatus'
# Expected: "ACTIVE"
```

**Definition of Done**:
- Table exists with correct schema
- Manual item insertion works
- No provisioned capacity (on-demand billing)

---

### Phase 3: Cognito User Pool Setup

**Scope**: Create Cognito User Pool with app client

**Files to Modify**:
- `template.yaml` - Add Cognito resources

**Template Addition**:
```yaml
Resources:
  CognitoUserPool:
    Type: AWS::Cognito::UserPool
    Properties:
      UserPoolName: SecureNotesUserPool
      AutoVerifiedAttributes:
        - email
      UsernameAttributes:
        - email
      Policies:
        PasswordPolicy:
          MinimumLength: 8
          RequireLowercase: true
          RequireNumbers: true
          RequireSymbols: false
          RequireUppercase: true

  CognitoUserPoolClient:
    Type: AWS::Cognito::UserPoolClient
    Properties:
      ClientName: SecureNotesClient
      UserPoolId: !Ref CognitoUserPool
      ExplicitAuthFlows:
        - ALLOW_USER_PASSWORD_AUTH
        - ALLOW_REFRESH_TOKEN_AUTH
      GenerateSecret: false
```

**Deliverables**:
- [ ] Cognito User Pool created
- [ ] App client configured for USER_PASSWORD_AUTH
- [ ] Can sign up test user via AWS CLI

**Verification**:
```bash
# Get outputs after deploy
aws cloudformation describe-stacks --stack-name secure-notes-app \
  --query 'Stacks[0].Outputs'

# Sign up test user
aws cognito-idp sign-up \
  --client-id <CLIENT_ID> \
  --username test@example.com \
  --password TestPass123!
```

**Definition of Done**:
- User pool exists with correct settings
- Can create user account
- Can obtain JWT token after confirmation

---

### Phase 4: Lambda CRUD Implementation

**Scope**: Implement all note operations in Lambda handler

**Files to Create/Modify**:
- `src/handlers/notes.py` - Main handler with all operations
- `template.yaml` - Connect Lambda to DynamoDB

**Handler Structure**:
```python
# src/handlers/notes.py

import json
import uuid
import boto3
from datetime import datetime

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('SecureNotes')

def handler(event, context):
    """Route to appropriate handler based on HTTP method and path."""
    http_method = event['httpMethod']
    path = event['path']
    user_id = event['requestContext']['authorizer']['claims']['sub']

    if http_method == 'POST' and path == '/notes':
        return create_note(event, user_id)
    elif http_method == 'GET' and path == '/notes':
        return list_notes(user_id)
    elif http_method == 'GET' and path.startswith('/notes/'):
        note_id = path.split('/')[-1]
        return get_note(user_id, note_id)
    elif http_method == 'PUT' and path.startswith('/notes/'):
        note_id = path.split('/')[-1]
        return update_note(event, user_id, note_id)
    elif http_method == 'DELETE' and path.startswith('/notes/'):
        note_id = path.split('/')[-1]
        return delete_note(user_id, note_id)

    return response(400, {'error': 'Invalid request'})

def create_note(event, user_id):
    body = json.loads(event['body'])
    note_id = str(uuid.uuid4())
    timestamp = datetime.utcnow().isoformat()

    item = {
        'userId': user_id,
        'noteId': note_id,
        'text': body['text'],
        'createdAt': timestamp,
        'updatedAt': timestamp
    }

    table.put_item(Item=item)
    return response(201, item)

def list_notes(user_id):
    result = table.query(
        KeyConditionExpression='userId = :uid',
        ExpressionAttributeValues={':uid': user_id}
    )
    return response(200, {'notes': result['Items']})

def get_note(user_id, note_id):
    result = table.get_item(Key={'userId': user_id, 'noteId': note_id})
    if 'Item' not in result:
        return response(404, {'error': 'Note not found'})
    return response(200, result['Item'])

def update_note(event, user_id, note_id):
    body = json.loads(event['body'])
    timestamp = datetime.utcnow().isoformat()

    result = table.update_item(
        Key={'userId': user_id, 'noteId': note_id},
        UpdateExpression='SET #text = :text, updatedAt = :ts',
        ExpressionAttributeNames={'#text': 'text'},
        ExpressionAttributeValues={':text': body['text'], ':ts': timestamp},
        ReturnValues='ALL_NEW',
        ConditionExpression='attribute_exists(userId)'
    )
    return response(200, result['Attributes'])

def delete_note(user_id, note_id):
    table.delete_item(Key={'userId': user_id, 'noteId': note_id})
    return response(204, None)

def response(status_code, body):
    return {
        'statusCode': status_code,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*'
        },
        'body': json.dumps(body) if body else ''
    }
```

**Deliverables**:
- [ ] All CRUD operations implemented
- [ ] Proper error handling
- [ ] Structured logging added
- [ ] IAM role with DynamoDB permissions

**Definition of Done**:
- All 5 endpoints return correct status codes
- User isolation works (can't access others' notes)
- Structured logs appear in CloudWatch

---

### Phase 5: API Gateway with Cognito Authorizer

**Scope**: Create API Gateway with JWT authorization

**Files to Modify**:
- `template.yaml` - Add API Gateway and authorizer

**Template Addition**:
```yaml
Resources:
  SecureNotesApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: prod
      Auth:
        DefaultAuthorizer: CognitoAuthorizer
        Authorizers:
          CognitoAuthorizer:
            UserPoolArn: !GetAtt CognitoUserPool.Arn
```

**Deliverables**:
- [ ] API Gateway deployed with Cognito authorizer
- [ ] All routes protected by JWT
- [ ] API URL output from stack

**Verification**:
```bash
# Without token - should fail
curl https://<api-id>.execute-api.<region>.amazonaws.com/prod/notes
# Expected: 401 Unauthorized

# With token - should succeed
curl -H "Authorization: Bearer $TOKEN" \
  https://<api-id>.execute-api.<region>.amazonaws.com/prod/notes
# Expected: 200 OK with notes array
```

**Definition of Done**:
- All endpoints reject requests without valid JWT
- Valid JWT allows access to user's notes only
- Cognito authorizer shows in API Gateway console

---

### Phase 6: Monitoring and Logging

**Scope**: Add CloudWatch alarms and structured logging

**Files to Modify**:
- `template.yaml` - Add CloudWatch resources
- `src/handlers/notes.py` - Add structured logging

**Structured Logging Addition**:
```python
import json

def log_event(event_type, user_id, note_id=None, **kwargs):
    print(json.dumps({
        'event': event_type,
        'userId': user_id,
        'noteId': note_id,
        **kwargs
    }))
```

**CloudWatch Alarms**:
```yaml
Resources:
  LambdaErrorAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: SecureNotesLambdaErrors
      MetricName: Errors
      Namespace: AWS/Lambda
      Statistic: Sum
      Period: 300
      EvaluationPeriods: 1
      Threshold: 1
      ComparisonOperator: GreaterThanOrEqualToThreshold
      Dimensions:
        - Name: FunctionName
          Value: !Ref NotesFunction
```

**Deliverables**:
- [ ] Structured JSON logs in CloudWatch
- [ ] Error alarm configured
- [ ] Throttle alarm configured
- [ ] API Gateway access logs enabled

**Definition of Done**:
- Logs show structured JSON format
- Alarms appear in CloudWatch console
- Can query logs with Insights

---

### Phase 7: Testing and Documentation

**Scope**: Write tests and finalize documentation

**Files to Create**:
- `tests/test_notes.py` - Unit tests
- `tests/test_integration.py` - Integration tests
- Update `README.md` with final documentation

**Test Structure**:
```python
# tests/test_notes.py

import pytest
from src.handlers.notes import create_note, response

def test_response_format():
    result = response(200, {'key': 'value'})
    assert result['statusCode'] == 200
    assert result['headers']['Content-Type'] == 'application/json'
    assert '"key": "value"' in result['body']

def test_response_empty_body():
    result = response(204, None)
    assert result['statusCode'] == 204
    assert result['body'] == ''
```

**Deliverables**:
- [ ] Unit tests for all handlers
- [ ] Integration tests with LocalStack or AWS
- [ ] README with curl examples
- [ ] Portfolio screenshots

**Definition of Done**:
- All tests pass locally
- README has complete curl examples
- Screenshots captured for portfolio

---

## Risk Assessment

| Risk | Likelihood | Impact | Score | Early Warning | Mitigation |
|------|------------|--------|-------|---------------|------------|
| AWS credentials misconfigured | High | High | 9 | `sam deploy` fails | Use `aws configure` wizard, verify with `aws sts get-caller-identity` |
| Cognito JWT validation fails | Medium | High | 6 | 401 on all requests | Test authorizer in API Gateway console first |
| DynamoDB permission denied | Medium | High | 6 | Lambda returns 500 | Check IAM role in CloudFormation, use `AmazonDynamoDBFullAccess` initially then restrict |
| Cold start latency too high | Medium | Medium | 4 | Slow first request | Keep Lambda under 50MB, consider provisioned concurrency later |
| Free Tier limits exceeded | Low | Medium | 3 | AWS billing alerts | Set up billing alarm at $5 threshold |
| CORS issues with web client | Medium | Low | 2 | Browser errors | Ensure Lambda returns CORS headers |

---

## Testing Strategy

### Test Pyramid

```
         ┌─────────────┐
         │   E2E (5%)  │  curl/Postman against deployed API
         ├─────────────┤
         │Integration  │  LocalStack or real AWS
         │   (25%)     │
         ├─────────────┤
         │   Unit      │  pytest with mocked boto3
         │   (70%)     │
         └─────────────┘
```

### Testing Framework

- **Framework**: pytest
- **Mocking**: moto (AWS service mocking) or unittest.mock
- **Coverage Target**: 80% for handlers

### First Three Tests to Write

1. **test_response_format** - Verify `response()` helper returns correct structure
2. **test_create_note_success** - Mock DynamoDB, verify item structure
3. **test_list_notes_empty** - Mock DynamoDB Query, verify empty array handling

```python
# tests/test_notes.py

import pytest
from unittest.mock import patch, MagicMock
import json

# Test 1: Response helper
def test_response_format():
    from src.handlers.notes import response
    result = response(200, {'message': 'ok'})

    assert result['statusCode'] == 200
    assert result['headers']['Content-Type'] == 'application/json'
    assert json.loads(result['body']) == {'message': 'ok'}

# Test 2: Create note
@patch('src.handlers.notes.table')
def test_create_note_success(mock_table):
    from src.handlers.notes import create_note

    event = {
        'body': json.dumps({'text': 'Test note'})
    }
    user_id = 'user-123'

    result = create_note(event, user_id)

    assert result['statusCode'] == 201
    mock_table.put_item.assert_called_once()
    body = json.loads(result['body'])
    assert body['text'] == 'Test note'
    assert body['userId'] == 'user-123'

# Test 3: List empty notes
@patch('src.handlers.notes.table')
def test_list_notes_empty(mock_table):
    from src.handlers.notes import list_notes

    mock_table.query.return_value = {'Items': []}

    result = list_notes('user-123')

    assert result['statusCode'] == 200
    body = json.loads(result['body'])
    assert body['notes'] == []
```

---

## First Concrete Task

### File to Create: `template.yaml`

**Location**: `/Users/sakeeb/Code repositories/secure-notes-app/template.yaml`

**Starter Code**:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Secure Notes App - Serverless API for managing personal notes

Globals:
  Function:
    Timeout: 10
    Runtime: python3.11
    MemorySize: 128

Resources:
  # Placeholder Lambda to verify deployment works
  NotesFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: SecureNotesHandler
      CodeUri: src/handlers/
      Handler: notes.handler
      Description: Handles all CRUD operations for secure notes
      Events:
        CreateNote:
          Type: Api
          Properties:
            Path: /notes
            Method: POST
        ListNotes:
          Type: Api
          Properties:
            Path: /notes
            Method: GET

Outputs:
  ApiUrl:
    Description: API Gateway endpoint URL
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/"
```

### Verification Steps

```bash
cd /Users/sakeeb/Code\ repositories/secure-notes-app

# Create handler placeholder
mkdir -p src/handlers
echo 'def handler(event, context): return {"statusCode": 200, "body": "OK"}' > src/handlers/notes.py

# Build
sam build

# Deploy (first time, use --guided)
sam deploy --guided
```

### First Commit Message

```
feat: Add SAM template with placeholder Lambda

- Initialize AWS SAM project structure
- Configure Lambda with API Gateway events
- Set up basic /notes endpoint scaffolding
```

---

## File Structure After Phase 1

```
secure-notes-app/
├── README.md
├── docs/
│   └── IMPLEMENTATION_PLAN.md
├── template.yaml
├── samconfig.toml (generated by sam deploy --guided)
├── src/
│   └── handlers/
│       ├── __init__.py
│       └── notes.py
├── tests/
│   ├── __init__.py
│   └── test_notes.py
└── requirements.txt
```

---

## Portfolio Deliverables Checklist

- [ ] Screenshot: Cognito User Pool and App Client
- [ ] Screenshot: API Gateway with Cognito Authorizer attached to method
- [ ] Screenshot: DynamoDB table with sample note items
- [ ] curl/Postman responses showing:
  - 201 Created (new note)
  - 200 OK (list notes)
  - 404 Not Found (invalid note ID)
- [ ] GitHub README with:
  - Problem statement
  - Architecture diagram
  - Setup steps
  - curl command examples
  - Enhancement roadmap
  - Cleanup instructions
