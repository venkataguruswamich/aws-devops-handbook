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

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('visitor-counter')

def lambda_handler(event, context):
    response = table.get_item(Key={'ID': '1'})
    
    if 'Item' in response:
        views = int(response['Item']['views'])
    else:
        views = 0

    views += 1

    table.put_item(Item={'ID': '1', 'views': views})

    return {
        'statusCode': 200,
        'headers': {
            'Access-Control-Allow-Origin': '*'
        },
        'body': json.dumps({'views': views})
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
async function updateCounter() {
    const response = await fetch("YOUR_LAMBDA_FUNCTION_URL");
    const data = await response.json();

    document.getElementById("counter-number").innerText = data.views;
}

updateCounter();
```

---

## 7.2 Update index.html

```html
<div id="counter-number">Loading...</div>
```

---

## 7.3 Upload Updated Files

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
