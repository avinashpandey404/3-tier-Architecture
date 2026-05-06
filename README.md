# AWS 3-Tier Architecture — Full Stack Cloud Project

> A production-style, end-to-end cloud application built on AWS using Free Tier resources.  
> **Frontend:** S3 + CloudFront &nbsp;|&nbsp; **Backend:** EC2 + Flask &nbsp;|&nbsp; **Database:** RDS MySQL

---

## 📸 Screenshots

> _Add your screenshots here after deployment_

| Frontend (CloudFront) | Flask API Response | CloudWatch Dashboard |
|---|---|---|
| `screenshots/frontend.png` | `screenshots/api.png` | `screenshots/cloudwatch.png` |

---

## 🏗️ Architecture Overview

```
                         ┌─────────────────────────────────────────────────────┐
                         │                   AWS Cloud                          │
                         │                                                       │
  User Browser           │   ┌─────────┐      ┌─────────────────────────────┐  │
      │                  │   │         │      │        Custom VPC            │  │
      │  HTTPS request   │   │ Amazon  │      │       10.0.0.0/16            │  │
      └─────────────────►│   │CloudFront├─────►                              │  │
                         │   │  (CDN)  │      │  ┌──────────────────────┐   │  │
                         │   └────┬────┘      │  │   Public Subnet      │   │  │
                         │        │           │  │   10.0.1.0/24        │   │  │
                         │        │           │  │                      │   │  │
                         │   ┌────▼────┐      │  │  ┌──────────────┐   │   │  │
                         │   │  Amazon │      │  │  │  EC2 Ubuntu  │   │   │  │
                         │   │   S3    │      │  │  │  t2.micro    │   │   │  │
                         │   │(Static  │      │  │  │  Flask :5000 │   │   │  │
                         │   │Website) │      │  │  └──────┬───────┘   │   │  │
                         │   └─────────┘      │  │         │           │   │  │
                         │                    │  │  ┌──────▼───────┐   │   │  │
                         │                    │  │  │  Amazon RDS  │   │   │  │
                         │                    │  │  │  MySQL 8.0   │   │   │  │
                         │                    │  │  │  db.t3.micro │   │   │  │
                         │                    │  │  └──────────────┘   │   │  │
                         │                    │  └──────────────────────┘   │  │
                         │                    └─────────────────────────────┘  │
                         │                                                       │
                         │   ┌─────────────────────┐   ┌─────────────────────┐ │
                         │   │   Amazon CloudWatch  │   │     IAM Roles       │ │
                         │   │  Metrics + Alarms    │   │   + Policies        │ │
                         │   └─────────────────────┘   └─────────────────────┘ │
                         └─────────────────────────────────────────────────────┘
```

**Traffic flow:**
1. User opens browser → hits CloudFront HTTPS URL
2. CloudFront serves cached HTML/CSS/JS from S3
3. JavaScript calls Flask API on EC2 (port 5000)
4. Flask queries RDS MySQL → returns JSON
5. JavaScript renders data in the browser

---

## ☁️ AWS Services Used

| Service | Tier | Purpose |
|---|---|---|
| **Amazon S3** | Free (5 GB) | Static website hosting for HTML/CSS/JS |
| **Amazon CloudFront** | Free (1 TB transfer) | CDN + HTTPS for the frontend |
| **Amazon EC2** | Free (t2.micro, 750 hrs/mo) | Ubuntu server running Flask API |
| **Amazon RDS** | Free (db.t3.micro, 750 hrs/mo) | Managed MySQL database |
| **Amazon VPC** | Free | Custom isolated network |
| **Amazon CloudWatch** | Free (basic metrics) | Monitoring, alarms, logs |
| **AWS IAM** | Free | Roles, policies, access control |

**Estimated monthly cost:** $0 (within Free Tier limits)

---

## 📁 Project Structure

```
aws-3tier-architecture/
│
├── frontend/
│   ├── index.html        # Main page — form + user list
│   ├── style.css         # Styling
│   └── script.js         # Fetch API calls to Flask backend
│
├── backend/
│   ├── app.py            # Flask application (routes, DB logic)
│   └── requirements.txt  # Python dependencies
│
├── iam/
│   └── ec2-cloudwatch-policy.json   # Custom IAM policy JSON
│
├── screenshots/          # Add your screenshots here
│
└── README.md
```

---

## 🚀 Setup Guide (Step-by-Step)

### Prerequisites

- AWS account (Free Tier)
- AWS CLI installed (optional)
- Git installed locally
- SSH client (Terminal on Mac/Linux, PowerShell or Git Bash on Windows)

---

### Phase 1 — AWS Account Setup

1. Create AWS account at [aws.amazon.com](https://aws.amazon.com)
2. Set a **billing alert** at $5/month (Billing → Budgets → Create Budget)
3. Create an **IAM admin user** (do not use root for daily work):
   - IAM → Users → Create User
   - Attach policy: `AdministratorAccess`
   - Download credentials CSV
4. Enable **MFA** on root account (strongly recommended)
5. Log in using the IAM user going forward

> **Root vs IAM user:** Root is the master account — never use it daily. IAM users have controllable, revokable access.

---

### Phase 2 — VPC & Networking

```
VPC CIDR:       10.0.0.0/16
Public Subnet:  10.0.1.0/24  (us-east-1a)
```

1. **Create VPC** → name: `MyProjectVPC`, CIDR: `10.0.0.0/16`
2. **Create Public Subnet** → `10.0.1.0/24`, enable auto-assign public IP
3. **Create Internet Gateway** → attach to VPC
4. **Create Route Table** → add route `0.0.0.0/0 → IGW` → associate with subnet

---

### Phase 3 — EC2 (Ubuntu)

**Launch settings:**

| Setting | Value |
|---|---|
| AMI | Ubuntu Server 22.04 LTS |
| Instance type | t2.micro (Free tier) |
| Network | MyProjectVPC / PublicSubnet-1a |
| Key pair | MyProjectKey.pem |

**Security Group rules:**

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| 22 | TCP | My IP only | SSH access |
| 80 | TCP | 0.0.0.0/0 | HTTP web traffic |
| 5000 | TCP | 0.0.0.0/0 | Flask API |

**Connect via SSH:**

```bash
# Mac / Linux / Git Bash
chmod 400 ~/MyProjectKey.pem
ssh -i ~/MyProjectKey.pem ubuntu@YOUR_EC2_PUBLIC_IP

# Windows PowerShell
icacls C:\Users\You\MyProjectKey.pem /inheritance:r
icacls C:\Users\You\MyProjectKey.pem /grant:r "%username%:R"
ssh -i C:\Users\You\MyProjectKey.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

---

### Phase 4 — Server Setup (Ubuntu)

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Python and tools
sudo apt install python3 python3-pip python3-venv -y

# Create project and virtual environment
mkdir ~/flask-app && cd ~/flask-app
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install flask mysql-connector-python flask-cors
pip freeze > requirements.txt
```

---

### Phase 5 — Flask Application

**`backend/app.py`** — complete source:

```python
from flask import Flask, request, jsonify
from flask_cors import CORS
import mysql.connector

app = Flask(__name__)
CORS(app)

DB_CONFIG = {
    'host':     'YOUR-RDS-ENDPOINT',
    'user':     'admin',
    'password': 'YOUR-DB-PASSWORD',
    'database': 'myappdb',
    'port':     3306
}

def get_db_connection():
    return mysql.connector.connect(**DB_CONFIG)

def init_db():
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS users (
            id INT AUTO_INCREMENT PRIMARY KEY,
            name VARCHAR(100) NOT NULL,
            email VARCHAR(150) NOT NULL UNIQUE,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    conn.commit()
    cursor.close()
    conn.close()

@app.route('/')
def health():
    return jsonify({'status': 'ok', 'message': 'Flask API running'})

@app.route('/users', methods=['GET'])
def get_users():
    try:
        conn = get_db_connection()
        cursor = conn.cursor(dictionary=True)
        cursor.execute('SELECT * FROM users ORDER BY id DESC')
        users = cursor.fetchall()
        cursor.close()
        conn.close()
        for u in users:
            if u.get('created_at'):
                u['created_at'] = str(u['created_at'])
        return jsonify({'users': users})
    except Exception as e:
        return jsonify({'error': str(e)}), 500

@app.route('/users', methods=['POST'])
def create_user():
    try:
        data = request.get_json()
        name  = data.get('name')
        email = data.get('email')
        if not name or not email:
            return jsonify({'error': 'name and email required'}), 400
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute(
            'INSERT INTO users (name, email) VALUES (%s, %s)',
            (name, email)
        )
        conn.commit()
        new_id = cursor.lastrowid
        cursor.close()
        conn.close()
        return jsonify({'message': 'User created', 'id': new_id}), 201
    except Exception as e:
        return jsonify({'error': str(e)}), 500

if __name__ == '__main__':
    init_db()
    app.run(host='0.0.0.0', port=5000, debug=True)
```

**`backend/requirements.txt`:**

```
Flask==3.0.0
flask-cors==4.0.0
mysql-connector-python==8.3.0
```

**Run the API:**

```bash
cd ~/flask-app && source venv/bin/activate
python3 app.py
```

**Test endpoints:**

```bash
# Health check
curl http://YOUR_EC2_IP:5000/

# Get all users
curl http://YOUR_EC2_IP:5000/users

# Create a user
curl -X POST http://YOUR_EC2_IP:5000/users \
  -H 'Content-Type: application/json' \
  -d '{"name": "Alice", "email": "alice@example.com"}'
```

---

### Phase 6 — RDS MySQL

**Create RDS instance:**

| Setting | Value |
|---|---|
| Engine | MySQL 8.0 |
| Template | Free tier |
| Instance class | db.t3.micro |
| Storage | 20 GB (no autoscaling) |
| DB name | myappdb |
| Username | admin |
| VPC | MyProjectVPC |
| Public access | Yes (learning only) |

**RDS Security Group** — inbound rule:

| Port | Source | Reason |
|---|---|---|
| 3306 (MySQL) | MyFlaskServerSG | Only EC2 can reach the DB |

> ⚠️ **Production note:** Never expose RDS publicly in production. Place it in a private subnet, reachable only from EC2 via security group rules.

**Update Flask config** with your RDS endpoint:

```python
DB_CONFIG = {
    'host': 'myproject-db.xxxx.us-east-1.rds.amazonaws.com',
    ...
}
```

---

### Phase 7 — IAM Role & Policies

**Custom CloudWatch Logs policy** (`iam/ec2-cloudwatch-policy.json`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudWatchLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Sid": "AllowCloudWatchMetrics",
      "Effect": "Allow",
      "Action": [
        "cloudwatch:PutMetricData",
        "cloudwatch:GetMetricStatistics",
        "cloudwatch:ListMetrics"
      ],
      "Resource": "*"
    }
  ]
}
```

**Create and attach role:**
1. IAM → Roles → Create Role → EC2 (trusted entity)
2. Attach `EC2-CloudWatch-Policy`
3. Name: `EC2-MyProject-Role`
4. EC2 → Actions → Security → Modify IAM Role → select role

> **IAM concepts:** A **User** is a person. A **Role** is an identity worn by an AWS service (no password). A **Policy** is the JSON document listing what's allowed.

---

### Phase 8 — Security Groups Summary

| Resource | Port | Source | Reason |
|---|---|---|---|
| EC2 | 22 | My IP | SSH — restricted to your machine only |
| EC2 | 80 | 0.0.0.0/0 | HTTP web traffic |
| EC2 | 5000 | 0.0.0.0/0 | Flask API access |
| RDS | 3306 | MyFlaskServerSG | DB access from EC2 only |

**Principle of least privilege:** Each resource is only open to exactly the traffic it needs — nothing more.

---

### Phase 9–10 — Frontend + S3 Hosting

**S3 bucket policy** (replace `BUCKET-NAME`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::BUCKET-NAME/*"
    }
  ]
}
```

**Setup steps:**
1. Create bucket → uncheck "Block all public access"
2. Enable static website hosting → index document: `index.html`
3. Apply the bucket policy above
4. Upload `index.html`, `style.css`, `script.js`

---

### Phase 11 — CloudFront CDN

1. CloudFront → Create Distribution
2. Origin: your S3 bucket
3. Viewer protocol: Redirect HTTP to HTTPS
4. Default root object: `index.html`
5. Wait ~10 minutes for deployment

Access your site at: `https://XXXX.cloudfront.net`

**To invalidate cache after updates:**

```
CloudFront → Invalidations → Create → Path: /*
```

---

### Phase 12 — Connect Frontend to Backend

Update `API_URL` in `frontend/script.js`:

```js
const API_URL = 'http://YOUR_EC2_PUBLIC_IP:5000';
```

CORS is already handled in Flask via `CORS(app)`. Re-upload `script.js` to S3 and invalidate CloudFront cache.

---

### Phase 13 — CloudWatch Monitoring

**Automatic metrics (no setup needed):**
- `CPUUtilization`
- `NetworkIn` / `NetworkOut`
- `StatusCheckFailed`

**Create CPU alarm:**
- CloudWatch → Alarms → Create Alarm
- Metric: `CPUUtilization > 80%` for 5 minutes
- Action: SNS email notification

---

## 🔐 Security Best Practices

| Practice | Status in This Project |
|---|---|
| No root user for daily operations | ✅ IAM user used |
| Billing alerts configured | ✅ $5 threshold |
| SSH restricted to My IP only | ✅ Security group rule |
| EC2 uses IAM Role (no hardcoded keys) | ✅ Role attached |
| RDS only reachable from EC2 SG | ✅ Security group chaining |
| S3 allows read-only (no write) publicly | ✅ GetObject only |
| MFA on root account | ✅ Recommended |
| Secrets not committed to Git | ✅ .gitignore |

**For production hardening (not in this project):**
- Place RDS in a private subnet (no public access)
- Use AWS Secrets Manager for DB passwords
- Add WAF in front of CloudFront
- Use HTTPS on EC2 (ACM certificate via ALB)
- Enable RDS automated backups and encryption at rest
- Use VPC Flow Logs for network auditing

---

## 💰 Cost Optimization

**Free Tier limits (per month):**

| Service | Free allowance |
|---|---|
| EC2 t2.micro | 750 hours |
| RDS db.t3.micro | 750 hours |
| S3 | 5 GB storage, 20,000 GET requests |
| CloudFront | 1 TB data transfer, 10M requests |
| CloudWatch | 10 custom metrics, 5 GB logs |

**Cost tips:**
- Stop EC2 when not using it (billing pauses for compute, not storage)
- Delete RDS when done — it charges even when idle
- Use CloudFront Price Class "North America & Europe" to reduce CDN costs
- Set a $5 billing alarm — you'll be notified before costs escalate
- After learning: terminate all resources (see cleanup below)

---

## 🧹 Cleanup (Avoid Charges)

Delete resources in this order:

```
1.  CloudFront → Disable → Delete distribution
2.  S3 → Empty bucket → Delete bucket
3.  EC2 → Terminate instance (not stop)
4.  RDS → Delete instance (uncheck final snapshot)
5.  EC2 Security Groups → delete custom ones
6.  Route Table → delete custom one
7.  Internet Gateway → detach → delete
8.  Subnets → delete
9.  VPC → delete
10. CloudWatch Alarms → delete
11. IAM Role → delete (optional)
12. SNS Topics → delete
```

> Verify in AWS Cost Explorer the next day that no ongoing charges appear.

---

## 🔧 Troubleshooting

| Problem | Likely Cause | Fix |
|---|---|---|
| SSH: `Connection timed out` | Port 22 not open in SG, or EC2 not in public subnet | Check security group, check subnet + IGW |
| SSH: `Permission denied (publickey)` | Wrong key file or wrong username | Ubuntu AMI = `ubuntu` user. Check `chmod 400` |
| Flask: `Can't connect to MySQL` | RDS SG blocking EC2, or wrong endpoint | Check SG allows 3306 from EC2 SG |
| Flask: `Access denied` for DB user | Wrong password in `DB_CONFIG` | Reset via RDS Console → Modify |
| Frontend: `CORS error` in browser | `CORS(app)` missing or Flask not running | Check Flask is running, check `flask-cors` installed |
| S3: `403 Forbidden` | Bucket policy missing or block public access still on | Re-apply bucket policy, verify block public access is off |
| CloudFront: shows old version | Cache not invalidated | Create invalidation with path `/*` |

---

## 🔮 Future Improvements

- [ ] Add Application Load Balancer (ALB) in front of EC2 for HTTPS on the API
- [ ] Move RDS to a private subnet (production-grade network isolation)
- [ ] Add Auto Scaling Group for EC2 (handle traffic spikes)
- [ ] Use AWS Secrets Manager for database credentials
- [ ] Add CI/CD pipeline with GitHub Actions → S3 deploy
- [ ] Containerize Flask with Docker → deploy to ECS Fargate
- [ ] Add user authentication with Amazon Cognito
- [ ] Enable RDS Multi-AZ for high availability
- [ ] Set up AWS WAF for DDoS and injection protection
- [ ] Add CloudFront custom domain with Route 53

---

## 👤 Author

**Your Name**  
[GitHub](https://github.com/yourusername) · [LinkedIn](https://linkedin.com/in/yourprofile)

Built as a hands-on AWS learning project covering core cloud services, networking, security, and full-stack deployment.

---

## 📄 License

MIT License — feel free to use this project as a learning template.
