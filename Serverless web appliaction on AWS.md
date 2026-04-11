#  🌐Serverless Web Application on AWS

[![AWS](https://img.shields.io/badge/AWS-Serverless-brightblue)](https://aws.amazon.com/serverless/)

## Project Overview
This project demonstrates how to build a fully serverless web application hosted on AWS. It hosts static website files (HTML, CSS, JavaScript) on S3 behind CloudFront CDN with custom domain/SSL, and implements a dynamic **view counter** using DynamoDB and Lambda. Every page load increments and displays the view count via a public Lambda Function URL.

![Image](https://user-images.githubusercontent.com/66474973/228492073-5cd3d975-3439-4ce4-b109-fb33997df3c3.png)
---

# 📌 Architecture

* **Amazon S3** – Static website hosting
* **Amazon CloudFront** – CDN + HTTPS
* **Amazon Route 53** – DNS management
* **AWS Certificate Manager (ACM)** – SSL/TLS certificate
* **AWS Lambda** – Backend logic
* **Amazon DynamoDB** – Stores visitor count
* **IAM** – Access control

---

# 🚀 Step-by-Step Implementation

---

# 🪣 Step 1: Create S3 Bucket & Upload Website

## 1.1 Create Bucket

1. Go to AWS Console → **S3**
2. Click **Create bucket**
3. Configure:

   * Bucket name → `your-unique-name` *(must be globally unique, lowercase only)*
   * Region → Choose nearest
4. Object Ownership → **ACLs disabled**
5. Block Public Access → ✅ Keep **ON**
6. Versioning → ❌ Disable
7. Default Encryption → ✅ Enable (SSE-S3)
8. Bucket Key → ✅ Enable
9. Click **Create bucket**

---

## 1.2 Upload Files

1. Open bucket → Click **Upload**
2. Upload:

   * `index.html`
   * `style.css`
   * `script.js`
3. Click **Upload**

---

# 🌍 Step 2: Configure CloudFront Distribution

## 2.1 Create Distribution

1. Go to **CloudFront**
2. Click **Create distribution**

### Origin Settings

* Origin domain → Select your S3 bucket
* Origin access → **Origin Access Control (OAC)**
* Create new OAC → Save

### Behavior Settings

* Viewer protocol policy → **Redirect HTTP to HTTPS**

### General Settings

* Default root object → `index.html`

3. Click **Create distribution**

---

## 2.2 Attach Bucket Policy

1. Open distribution → **Origins tab**
2. Click **Edit → Copy bucket policy**
3. Go to:

   * S3 → Bucket → **Permissions**
4. Paste policy in **Bucket Policy**
5. Save

---

# 🔐 Step 3: Custom Domain, SSL & DNS

---

## 3.1 Create Hosted Zone (Route 53)

1. Go to **Route 53**
2. Click **Hosted Zones → Create hosted zone**
3. Enter domain → `gurusaidevops.com`
4. Copy **NS records**

---

## 3.2 Update Domain Registrar

1. Go to domain provider (GoDaddy, Namecheap)
2. Replace nameservers with Route 53 NS
3. Save changes
   ⏳ Wait for DNS propagation (up to 24 hrs)

---

## 3.3 Request SSL Certificate

1. Go to **ACM (us-east-1 region)**
2. Click **Request certificate**
3. Add:

   * `gurusaidevops.com`
   * `*.gurusaidevops.com`
4. Select **DNS validation**
5. Click **Request**

---

## 3.4 Validate Certificate

1. Click **Create records in Route 53**
2. Wait until status = ✅ **Issued**

---

## 3.5 Attach SSL to CloudFront

1. Go to CloudFront → Distribution → **Edit**
2. Add:

   * Alternate domain (CNAME): `gurusaidevops.com`
3. Select SSL certificate
4. Save

---

## 3.6 Create DNS Record

1. Go to Route 53 → Hosted Zone
2. Click **Create record**
3. Configure:

   * Type → **A**
   * Enable **Alias**
   * Target → CloudFront distribution
4. Save

---

# 🗄️ Step 4: DynamoDB Setup

## 4.1 Create Table

1. Go to **DynamoDB**
2. Click **Create table**
3. Configure:

   * Table name → `visitor-counter`
   * Partition key → `ID` (String)
4. Click **Create table**

---

## 4.2 Insert Initial Data

1. Open table → **Explore items**
2. Click **Create item**
3. Add:

```json
{
  "ID": "1",
  "views": 0
}
```

4. Save

---

# 🔐 Step 5: Create IAM Role

1. Go to **IAM**
2. Click **Roles → Create role**
3. Select:

   * Trusted entity → **Lambda**
4. Attach policy:

   * `AmazonDynamoDBFullAccess`
5. Role name → `lambda-dynamodb-role`
6. Click **Create role**

---

# ⚙️ Step 6: Create AWS Lambda Function

---

## 6.1 Create Function

1. Go to **Lambda**
2. Click **Create function**
3. Select:

   * Author from scratch
4. Configure:

   * Name → `visitor-counter-function`
   * Runtime → Python 3.8
5. Click **Create**

---

## 6.2 Attach IAM Role

1. Go to **Configuration → Permissions**
2. Click **Edit**
3. Select:

   * `lambda-dynamodb-role`
4. Save

---

## 6.3 Enable Function URL

1. Go to **Function URL**
2. Click **Create**
3. Auth type → **NONE**
4. Save and copy URL

---

## 6.4 Add Lambda Code

```python
import json
import boto3
from datetime import datetime, timedelta

dynamodb = boto3.resource('dynamodb', region_name='ap-south-0')
table = dynamodb.Table("Mention the S3 Bucket name")

def lambda_handler(event, context):
    try:
        now = datetime.utcnow()
        today_str = now.strftime('%Y-%m-%d')
        hour_str = now.strftime('%H:00')

        # MAIN COUNTER
        response = table.get_item(Key={'id': '1'})
        current_views = int(response.get('Item', {}).get('views', 0))
        new_views = current_views + 1

        table.put_item(Item={
            'id': '1',
            'views': new_views,
            'last_updated': now.isoformat()
        })

        # DAILY
        daily_response = table.get_item(Key={'id': f'daily_{today_str}'})
        daily_views = int(daily_response.get('Item', {}).get('views', 0))

        table.put_item(Item={
            'id': f'daily_{today_str}',
            'views': daily_views + 1,
            'date': today_str
        })

        # WEEKLY
        week_views = 0
        for i in range(7):
            day = (now - timedelta(days=i)).strftime('%Y-%m-%d')
            day_response = table.get_item(Key={'id': f'daily_{day}'})
            week_views += int(day_response.get('Item', {}).get('views', 0))

        return {
            'statusCode': 200,
            'headers': {
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'views': new_views,
                'today': daily_views + 1,
                'week': week_views,
                'peak_hour': hour_str,
                'fun_fact': f"You're visitor #{new_views} 🎉"
            })
        }

    except Exception as e:
        return {
            'statusCode': 500,
            'headers': {
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'error': str(e)
            })
        }
```

5. Click **Deploy**

---

## 6.5 Test Function

1. Click **Test**
2. Run test
3. Confirm counter increments

---

# 🔗 Step 7: Connect Frontend
---
## 7.1 Update script.js

```javascript
const API_URL = "YOUR_LAMBDA_FUNCTION_URL"; // ⚠️ REPLACE WITH YOUR ACTUAL URL

function setText(id, value) {
    const el = document.getElementById(id);
    if (el) {
        el.innerText = value;
    } else {
        console.warn("Missing element:", id);
    }
}

async function updateCounter() {
    try {
        setText("status", "Loading...");

        const response = await fetch(API_URL);
        const data = await response.json();

        if (!response.ok) {
            throw new Error(data.error || "API Error");
        }

        // Update UI safely
        setText("views", data.views);
        setText("today", data.today);
        setText("week", data.week);
        setText("peak", data.peak_hour);
        setText("fun", data.fun_fact);

        setText("status", "✅ Updated successfully");

    } catch (error) {
        console.error("Fetch error:", error);

        setText("views", "⚠️ Error");
        setText("status", "❌ Failed to update counter");
    }
}

// Auto run on page load
updateCounter();
```
---

## 7.2 Update style.css

```style.css
body {
  margin: 0;
  font-family: 'Segoe UI', sans-serif;
  background: linear-gradient(135deg, #0f172a, #1e293b);
  color: #fff;
}

.container {
  max-width: 600px;
  margin: 50px auto;
  text-align: center;
}

h1 {
  margin-bottom: 20px;
}

.card {
  background: #111827;
  padding: 20px;
  border-radius: 15px;
  margin-bottom: 20px;
}

.big-number {
  font-size: 60px;
  color: #f87171;
}

.stats {
  display: flex;
  justify-content: space-between;
  gap: 10px;
}

.box {
  background: #1f2937;
  flex: 1;
  padding: 15px;
  border-radius: 10px;
}

.fun-fact {
  margin-top: 20px;
  font-style: italic;
}

button {
  margin-top: 20px;
  padding: 10px 20px;
  border: none;
  background: #6366f1;
  color: white;
  border-radius: 8px;
  cursor: pointer;
}

button:hover {
  background: #4f46e5;
}

.status {
  margin-top: 15px;
  font-size: 14px;
}

.ok {
  color: #22c55e;
  margin: 0 10px;
}
```

---
---

## 7.3 Upload Updated index.htmi

```index.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Serverless Visitor Counter</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">

    <!-- Optional favicon (fix 403 warning) -->
    <link rel="icon" href="favicon.ico">

    <style>
        body {
            font-family: Arial;
            text-align: center;
            background: #0f172a;
            color: white;
        }
        .card {
            background: #1e293b;
            padding: 20px;
            margin: 20px auto;
            width: 300px;
            border-radius: 10px;
        }
        h2 {
            font-size: 40px;
        }
    </style>
</head>

<body>

<h1>⚡ Serverless on AWS</h1>
<h2>Visitor Counter</h2>

<div class="card">
    <h2 id="views">--</h2>
    <p>powered by Lambda + DynamoDB</p>
</div>

<div class="card">
    <p>📊 Today: <span id="today">--</span></p>
    <p>📅 This Week: <span id="week">--</span></p>
    <p>⏰ Peak Hour: <span id="peak">--</span></p>
</div>

<div class="card">
    <p id="fun">Loading fun fact...</p>
</div>

<p id="status">Loading...</p>

<button onclick="updateCounter()">🔄 Refresh</button>

<script src="script.js"></script>

</body>
</html>
```
---
## 7.4 Upload Updated Files

1. Go to S3
2. Upload:

   * `index.html`
   * `script.js`
3. Overwrite existing files

---

# 🔄 Step 8: Invalidate CloudFront Cache

1. Go to **CloudFront**
2. Open distribution
3. Go to **Invalidations**
4. Create invalidation:

   * Path → `/*`
5. Submit

---

# ✅ Step 9: Final Testing

1. Open browser
2. Visit:

   * `https://gurusaidevops.com`
3. Refresh page
4. ✅ Visitor counter should increase

---

# 🛠️ Troubleshooting

| Issue                | Solution                                 |
| -------------------- | ---------------------------------------- |
| Site not loading     | Check CloudFront status = Deployed       |
| Access denied        | Verify S3 bucket policy + OAC            |
| SSL not working      | Ensure ACM cert is issued in us-east-1   |
| Domain not working   | Check Route 53 NS records                |
| Counter not updating | Verify Lambda URL + DynamoDB             |
| CORS error           | Add `Access-Control-Allow-Origin` header |
| Old content showing  | Invalidate CloudFront cache              |

---

# 🧹 Cleanup (Avoid Charges)

⚠️ Delete resources in this order:

1. **CloudFront**

   * Disable → Delete distribution

2. **Route 53**

   * Delete A record
   * Delete hosted zone

3. **ACM**

   * Delete certificate

4. **Lambda**

   * Delete function

5. **DynamoDB**

   * Delete table

6. **IAM**

   * Delete role

7. **S3**

   * Empty bucket → Delete bucket

---

# 💰 Cost Estimation

## Free Tier

* S3 → 5GB storage
* Lambda → 1M requests
* DynamoDB → 25GB
* CloudFront → Free tier usage
* Route 53 → Not free (~$0.50/month)

---

## Estimated Cost

| Usage            | Cost           |
| ---------------- | -------------- |
| Beginner project | $0 – $2/month  |
| Medium traffic   | $5 – $15/month |

---

# ⚠️ Cost Optimization Tips

* Use CloudFront caching
* Delete unused resources
* Monitor AWS Billing Dashboard
* Keep S3 lifecycle rules

---

# 🎯 Final Outcome

✔ Fully serverless architecture
✔ Secure HTTPS website
✔ Custom domain
✔ Dynamic visitor counter
✔ Low cost & scalable

---

# 🙌 Conclusion

You have successfully built a **production-ready AWS serverless web application** using modern cloud architecture.

---
## 📬 Author

VenkataGuruSwami.ch