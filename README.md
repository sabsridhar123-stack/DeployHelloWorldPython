## 🔹 Part 1: Prerequisites (Local Setup on Windows, PowerShell)
 
1. Check Python installation
 
   python --version

   * Should be 3.8+
   * If missing: download from [Python.org](https://www.python.org/downloads/)
 
2. Install pip & virtualenv
 
   python -m pip install --upgrade pip
   pip install virtualenv
   
3. Check Git
   
   git --version
   
   * If missing: Install from [Git-scm.com](https://git-scm.com/download/win)
 
4. Install AWS CLI
 
   * Download installer: [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
   * After install:
   aws --version
     
   * Configure:
     
     aws configure
     
     Provide:
 
     * Access Key ID
     * Secret Key
     * Default region: `us-east-1` (or your choice)
     * Output: `json`
 
---
 
## 🔹 Part 2: Phase 1 (Local Development - Hello World App)
 
### Step 1: Create project
 

mkdir aws-hello-world
cd aws-hello-world
 
# Create virtual environment
python -m venv venv
.\venv\Scripts\Activate
 
# Basic structure
mkdir app,templates,static
ni app\__init__.py
ni app\routes.py
ni requirements.txt
ni app.py

 
### Step 2: Install dependencies
 

pip install flask boto3
pip freeze > requirements.txt

 
### Step 3: Write code
 
app.py
 
python
from flask import Flask
 
app = Flask(__name__)
 
@app.route("/")
def home():
    return "Hello World from Flask on AWS!"
 
@app.route("/health")
def health():
    return {"status": "ok"}
 
if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)

 
### Step 4: Run & test locally
 

python app.py

 
* Open browser: `http://127.0.0.1:5000/`
* Should see: Hello World from Flask on AWS!
* Test health endpoint: `http://127.0.0.1:5000/health`
 
---
 
## 🔹 Part 3: Phase 2 (AWS Infra via Console)
 
We’ll now prep AWS infra for later deployment.
 
---
 
### Step 1: IAM
 
#### 1.1 Create IAM Role for EC2
 
1. Go to AWS Console → IAM → Roles → Create role
2. Trusted entity: EC2
3. Attach policies:
 
   * `AmazonS3FullAccess`
   * `CloudWatchAgentServerPolicy`
4. Role name: `HelloWorldEC2Role`
 
---
 
### Step 2: VPC
 
#### 2.1 Create VPC
 
1. VPC → Create VPC
 
   * Name: `HelloWorld-VPC`
   * CIDR: `10.0.0.0/16`
 
#### 2.2 Create Public Subnet
 
* Name: `HelloWorld-Public-Subnet`
* VPC: `HelloWorld-VPC`
* AZ: `us-east-1a`
* CIDR: `10.0.1.0/24`
 
#### 2.3 Internet Gateway
 
* Create IGW: `HelloWorld-IGW`
* Attach to `HelloWorld-VPC`
* Update route table:
 
  * Destination: `0.0.0.0/0`
  * Target: `HelloWorld-IGW`
 
#### 2.4 Security Group
 
* Name: `HelloWorld-Web-SG`
* VPC: `HelloWorld-VPC`
* Inbound rules:
 
  * SSH: 22 (Your IP)
  * HTTP: 80 (Anywhere)
  * Custom TCP: 5000 (Anywhere, for Flask testing)
 
---
 
### Step 3: S3 Bucket
 
1. Go to S3 → Create bucket
 
   * Name: `hello-world-uploads-[unique-id]`
   * Region: same as EC2
   * Block public access: Keep ON
   * Enable versioning
   * Enable encryption
 
2. Bucket policy (for EC2 role):
 
json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowEC2Access",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::YOUR-ACCOUNT-ID:role/HelloWorldEC2Role"
      },
      "Action": ["s3:GetObject","s3:PutObject","s3:DeleteObject"],
      "Resource": "arn:aws:s3:::hello-world-uploads-[unique-id]/*"
    }
  ]
}

 
---
 
### Step 4: EC2 Instance
 
1. Launch instance
 
   * Name: `HelloWorld-EC2`
   * AMI: Amazon Linux 2023
   * Type: t2.micro
   * Key Pair: create/download (`.pem` file)
   * Network: `HelloWorld-VPC`
   * Subnet: `HelloWorld-Public-Subnet`
   * Auto-assign Public IP: Enable
   * Security Group: `HelloWorld-Web-SG`
   * IAM Role: `HelloWorldEC2Role`
   * Storage: 8GB gp3
 
2. Connect
 
   ssh -i your-key.pem ec2-user@<EC2-Public-IP>