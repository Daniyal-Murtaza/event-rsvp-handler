# Event RSVP Handler

A full-stack serverless web app where users can browse upcoming events and RSVP (yes/no) in real time. Built entirely on AWS.

## Architecture

```
Browser
  │
  ▼
Amazon CloudFront (CDN)
  │
  ▼
Amazon S3 (static frontend: HTML, CSS, JS)
  │ (JS makes API calls)
  ▼
Amazon API Gateway (HTTP API)
  │
  ▼
AWS Lambda (Node.js — index.js)
  │               │
  ▼               ▼
Amazon RDS     Amazon DynamoDB
(MySQL)        (NoSQL)
Event details  RSVP responses + counts
```

## Tech Stack

| Layer     | Service                        | Purpose                              |
|-----------|--------------------------------|--------------------------------------|
| Frontend  | Amazon S3 + CloudFront         | Host and serve static files globally |
| API       | Amazon API Gateway (HTTP API)  | Route HTTP requests to Lambda        |
| Backend   | AWS Lambda (Node.js ESM)       | Business logic                       |
| Database  | Amazon RDS MySQL               | Structured event data                |
| Database  | Amazon DynamoDB                | RSVP responses and live counts       |
| Network   | VPC + Security Groups          | Private networking for RDS           |
| Network   | VPC Endpoint (DynamoDB)        | Let Lambda reach DynamoDB inside VPC |

## Project Files

```
event-rsvp-handler/
├── index.js              # Lambda function (backend)
├── package.json          # Backend dependencies
├── index.html            # Frontend structure
├── style.css             # Frontend styles
├── utils.js              # API base URL + shared helpers
├── events.js             # Event/RSVP API calls and DOM logic
├── app.js                # App entry point (initializes on page load)
├── database-notes.txt    # SQL to create table and insert sample events
└── banner/               # Event banner images (uploaded to S3)
```

---

## Prerequisites

Install these before starting:

- [Node.js](https://nodejs.org/) — to install dependencies and zip the Lambda package locally
- [VS Code](https://code.visualstudio.com/) or any code editor
- [DBeaver](https://dbeaver.io/) or MySQL Workbench — to connect to RDS and run SQL
- [Postman](https://www.postman.com/) or Thunder Client — to test API endpoints
- Git + GitHub (optional, recommended for portfolio)

---

## AWS Setup (Step by Step)

### 1. Set Your Region

Use **N.Virginia US** for all services, or pick one region and stay consistent throughout.

---

### 2. Create a Budget Alarm

In the AWS Console, search **Budgets** → Create budget → Monthly cost budget.

- Set amount: `$10` (or lower)
- Add your email for notifications at 85% and 100% spend

---

### 3. Amazon RDS (MySQL)

**Console → RDS → Create database**

| Setting               | Value                        |
|-----------------------|------------------------------|
| Engine                | MySQL                        |
| Template              | Free tier                    |
| DB instance identifier| `event-rsvp`                 |
| Master username       | `admin`                      |
| Instance class        | `db.t3.micro`                |
| Availability zone     | Single AZ                    |
| Public access         | Yes                          |
| Initial database name | `eventsDB`                   |

After creation:
1. Go to the RDS instance → **VPC security group** → **Edit inbound rules**
2. Add rule: **All traffic**, Source = **My IP**

**Connect with DBeaver:**
- Host: your RDS endpoint (from the console)
- Port: `3306`
- Database: `eventsDB`
- Username / Password: what you set above

**Create the events table** (also in `database-notes.txt`):

```sql
CREATE TABLE events (
  event_id      VARCHAR(64) PRIMARY KEY,
  title         VARCHAR(200) NOT NULL,
  description   TEXT,
  start_at      DATETIME NOT NULL,
  venue         VARCHAR(200),
  banner_url    VARCHAR(500),
  created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Insert sample events** (also in `database-notes.txt`):

```sql
INSERT INTO events (event_id, title, description, start_at, venue, banner_url)
VALUES
(
  'aws-meetup-manila-2025',
  'AWS User Group Manila Monthly Meetup 2025',
  'Join fellow builders for lightning talks, live demos, and networking with AWS enthusiasts in Manila.',
  '2025-10-18 18:00:00',
  'AWS PH Office, Bonifacio Global City',
  ''
),
(
  'aws-buildercards-tournament-2025',
  'AWS BuilderCards Tournament 2025',
  'Learn all about AWS BuilderCards.',
  '2025-11-08 17:00:00',
  'AWS PH Office, Bonifacio Global City',
  ''
),
(
  'aws-community-day-ph-2025',
  'AWS Community Day Philippines 2025',
  'The biggest gathering of AWS builders, user group leaders, and cloud enthusiasts in the Philippines – featuring talks, demos, and community showcases.',
  '2025-11-30 09:00:00',
  'AWS PH Office, Bonifacio Global City',
  ''
);
```

> **Tip:** Stop the RDS instance (Actions → Stop temporarily) when not using it to avoid idle charges.

---

### 4. Amazon DynamoDB

**Console → DynamoDB → Create table**

| Setting       | Value                   |
|---------------|-------------------------|
| Table name    | `event-rsvp-responses`  |
| Partition key | `pk` (String)           |
| Sort key      | `sk` (String)           |

No further configuration needed. Leave it running — idle cost is near zero.

**Key design:**

| Item type        | pk                    | sk                        |
|------------------|-----------------------|---------------------------|
| Attendee record  | `EVENT#<event_id>`    | `RESPONDENT#<email>`      |
| Yes count        | `EVENT#<event_id>`    | `RESPONSE#Yes`            |
| No count         | `EVENT#<event_id>`    | `RESPONSE#No`             |

---

### 5. AWS Lambda

**Console → Lambda → Create function**

| Setting                  | Value                    |
|--------------------------|--------------------------|
| Function name            | `event-rsvp-handler`     |
| Runtime                  | Node.js (latest LTS)     |
| Architecture             | x86_64                   |
| Execution role           | Create new with basic permissions |

**After creation:**

**General configuration → Edit timeout:** Set to `30 seconds` (default 3s is too short for RDS cold start).

**Environment variables → Edit → Add:**

| Key       | Value                        |
|-----------|------------------------------|
| DB_HOST   | Your RDS endpoint            |
| DB_USER   | `admin`                      |
| DB_PASS   | Your RDS password            |
| DB_NAME   | `eventsDB`                   |
| REGION    | `ap-southeast-1`             |

**Permissions → click the role name → Attach policies:**

- `AWSLambdaVPCAccessExecutionRole` — lets Lambda connect to RDS inside the VPC
- `AmazonDynamoDBFullAccess` — lets Lambda read/write DynamoDB

**Configuration → VPC → Edit:**

Attach Lambda to the **same VPC, subnets, and security group** as your RDS instance.

**Security group inbound rule (for Lambda ↔ RDS):**

Go to the shared security group → Edit inbound rules → Add:
- Type: `MySQL/Aurora`, Port: `3306`, Source: the security group ID itself (self-referencing)

**Deploy the Lambda code:**

```bash
# In the project root
npm install
zip -r rsvp-backend.zip .
```

Upload `rsvp-backend.zip` via **Lambda → Upload from → .zip file**.

In **Runtime settings**, confirm the handler is set to: `index.handler`

> If you edit code directly in the Lambda console (quick bug fix), click **Deploy** to apply changes without re-zipping.

---

### 6. VPC Endpoint for DynamoDB

Lambda runs inside a VPC but DynamoDB is a public AWS service. Without an endpoint, Lambda cannot reach DynamoDB from inside the VPC.

**Console → VPC → Endpoints → Create endpoint**

| Setting     | Value                          |
|-------------|--------------------------------|
| Service     | `com.amazonaws.<region>.dynamodb` |
| VPC         | Same VPC as Lambda and RDS     |
| Route table | Select the main route table    |
| Policy      | Full access                    |

---

### 7. Amazon API Gateway

**Console → API Gateway → Create API → HTTP API**

| Setting    | Value                        |
|------------|------------------------------|
| Name       | `event-rsvp-api`             |
| Integration| Lambda — `event-rsvp-handler`|

**Add these routes:**

| Method | Path                       |
|--------|----------------------------|
| GET    | `/events`                  |
| GET    | `/event/{event_id}`        |
| GET    | `/stats/{event_id}`        |
| GET    | `/attendees/{event_id}`    |
| POST   | `/rsvp`                    |

**CORS → Configure:**

- Allow origins: `*`
- Allow methods: `GET, POST, OPTIONS`
- Allow headers: `Content-Type, Authorization, X-Api-Key, X-Amz-Date, X-Amz-Security-Token, X-Requested-With`

Copy the **default endpoint URL** — you'll need it in the next step.

---

### 8. Connect the Frontend to the Backend

Open `utils.js` and update the API base URL:

```js
const API_BASE_URL = 'https://<your-api-gateway-id>.execute-api.<region>.amazonaws.com/';
```

---

### 9. Test APIs with Postman

Use your API Gateway endpoint URL as the base.

| Test                    | Method | Path                                   | Body (JSON)                                                         |
|-------------------------|--------|----------------------------------------|---------------------------------------------------------------------|
| List all events         | GET    | `/events`                              | —                                                                   |
| Get one event           | GET    | `/event/aws-meetup-manila-2025`        | —                                                                   |
| Submit RSVP             | POST   | `/rsvp`                                | `{"event_id":"...","full_name":"...","email":"...","response":"Yes"}` |
| Get yes/no counts       | GET    | `/stats/aws-meetup-manila-2025`        | —                                                                   |
| Get attendees           | GET    | `/attendees/aws-meetup-manila-2025`    | —                                                                   |

Check **CloudWatch → Log groups → /aws/lambda/event-rsvp-handler** for Lambda logs if something fails.

---

### 10. Amazon S3

**Console → S3 → Create bucket**

| Setting              | Value                          |
|----------------------|--------------------------------|
| Bucket name          | Any unique name                |
| Region               | Same as other services         |
| Block public access  | Keep ON (CloudFront serves it) |

Upload all frontend files:
- `index.html`, `style.css`, `utils.js`, `events.js`, `app.js`
- Create a `banner/` folder and upload your banner images

---

### 11. Amazon CloudFront

**Console → CloudFront → Create distribution**

| Setting                   | Value                               |
|---------------------------|-------------------------------------|
| Origin domain             | Your S3 bucket                      |
| S3 bucket access          | Yes — restrict to CloudFront only   |
| Default root object       | `index.html`                        |

After deployment, copy the **Distribution domain name** (e.g. `d1abc123.cloudfront.net`).

**Update banner URLs in RDS:**

```sql
UPDATE events
SET banner_url = 'https://d1abc123.cloudfront.net/banner/your-image.jpg'
WHERE event_id = 'aws-meetup-manila-2025';
```

Your site is now live at the CloudFront URL.

---

### 12. Updating Frontend Files (CloudFront Cache Invalidation)

When you change any HTML, CSS, or JS and re-upload to S3, CloudFront may still serve the old cached version. You need to invalidate the cache:

1. Go to **CloudFront → your distribution → Invalidations → Create invalidation**
2. Enter `/*` to clear the entire cache
3. Wait for the status to show **Completed**
4. Hard refresh your browser:
   - **Windows:** `Ctrl + F5`
   - **Mac:** `Cmd + Shift + R`

---

## Running Locally

```bash
# Serve the frontend locally (Python 3)
python3 -m http.server 8000
# Open http://localhost:8000
```

Make sure `utils.js` has your API Gateway URL set — the local frontend still calls the real Lambda backend.

---

## API Reference

All endpoints are relative to your API Gateway base URL.

| Method | Path                        | Description                           |
|--------|-----------------------------|---------------------------------------|
| GET    | `/events`                   | List all events (sorted by date)      |
| GET    | `/event/{event_id}`         | Get details of a single event         |
| GET    | `/stats/{event_id}`         | Get yes/no RSVP counts                |
| GET    | `/attendees/{event_id}`     | Get list of attendees                 |
| POST   | `/rsvp`                     | Submit an RSVP                        |

**POST /rsvp body:**
```json
{
  "event_id": "aws-meetup-manila-2025",
  "full_name": "Jane Doe",
  "email": "jane@example.com",
  "response": "Yes"
}
```

---

## Cost Notes

| Service      | Free tier                          | After free tier             |
|--------------|------------------------------------|-----------------------------|
| RDS          | 750 hrs/month (t3.micro), 1 year   | ~$0–$5/month if left running|
| DynamoDB     | 25 GB + 25 WCU/RCU free forever    | Near $0 for small projects  |
| Lambda       | 1M requests/month free forever     | Near $0 for demos           |
| API Gateway  | 1M calls/month (first year)        | Near $0 for demos           |
| S3           | 5 GB storage, 20k GETs (1 year)    | ~$0.023/GB                  |
| CloudFront   | 1 TB transfer/month (1 year)       | ~$0.085/GB                  |

---

## Pause (Keep Data, Stop Charges)

1. **RDS** → Actions → Stop temporarily
2. **CloudFront** → Disable distribution
3. **Lambda / DynamoDB / API Gateway** — no action needed at idle

---

## Full Teardown (Zero Ongoing Cost)

Do this in order:

1. Export DynamoDB table to S3 (DynamoDB → Actions → Export to S3)
2. Take RDS snapshot (RDS → Actions → Take snapshot), or skip for a simple demo
3. Delete CloudFront distribution (must disable first, then delete)
4. Delete API Gateway
5. Delete Lambda function and its IAM role
6. Delete RDS instance
7. Empty and delete S3 bucket
8. Delete custom security groups and VPC endpoints (default VPC/security group cannot be deleted — that's fine)
9. Delete CloudWatch log groups manually if needed

> Keep your Budget alarm active even after teardown so you get notified of any leftover charges.
