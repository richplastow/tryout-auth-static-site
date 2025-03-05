# tryout-auth-static-site

> An example of a simple static (CDN hosted) site, with some password-protected pages

- Version: 0.0.0
- Created: 4th March 2025
- Updated: 5th March 2025
- Author: Rich Plastow
- License: MIT
- Repo: <https://github.com/richplastow/tryout-auth-static-site>
- Hosted on GitHub Pages: <https://richplastow.com/tryout-auth-static-site/>
- Hosted as an AWS S3 website:
  <http://tryout-auth-static-site-s3-bucket.s3-website-us-east-1.amazonaws.com>

# Tryout 1: AWS S3 with CloudFront, using Lambda@Edge for authentication

## Step 1.1: Install the AWS CLI

Check whether you already have the AWS CLI installed by running:

```bash
which aws # aws not found
aws --version # zsh: command not found: aws
```

Install the AWS CLI by following the instructions at
<https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html>.

For example, to install on macOS for all of your Mac's users (assuming you have
admin privileges on your Mac):

```bash
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
#   % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
#                                  Dload  Upload   Total   Spent    Left  Speed
# 100 39.1M  100 39.1M    0     0  5002k      0  0:00:08  0:00:08 --:--:-- 5531k
# Password:
# installer: Package name is AWS Command Line Interface
# installer: Installing at base path /
# installer: The install was successful.
which aws # /usr/local/bin/aws
aws --version # aws-cli/2.24.16 Python/3.12.9 Darwin/21.6.0 exe/x86_64
```

## Step 1.2: Create your ‘Access Key’ and ‘Secret Access Key’

Visit <https://us-east-1.console.aws.amazon.com/iam/home>, scroll down to the
‘Quick links’ section, and click the ‘My security credentials’ link.

Scroll to the ‘Access keys (0)’ section, click the ‘Create access key’ button,
note the “Root user access keys are not recommended” warning, tick the
“I understand creating a root access key is not a best practice ...” checkbox,
and click the ‘Create access key’ button.

Copy-paste the ‘Access Key’ and ‘Secret Access Key’ into a password manager, and
click ‘Done’.

TODO: show best practice, by avoiding using a root access key

## Step 1.3: Configure the AWS CLI with your credentials

```bash
aws iam list-policies
# Unable to locate credentials. You can configure credentials by running "aws configure".
aws configure
# AWS Access Key ID [None]: AKI**************4XJ
# AWS Secret Access Key [None]: oUl**********************************VqU
# Default region name [None]: eu-central-1
# Default output format [None]: json
cat ~/.aws/config
# [default]
# region = eu-central-1
# output = json
cat ~/.aws/credentials # you can rename to ~/.aws/credentials_HID to temporarily disable access
# [default]
# aws_access_key_id = AKI**************4XJ
# aws_secret_access_key = oUl**********************************VqU
aws sts get-session-token
# {
#     "Credentials": {
#         "AccessKeyId": "AKI**************4XJ",
#         "SecretAccessKey": "StZ**********************************27w",
#         "SessionToken": "...",
#         "Expiration": "2025-03-... [in 1 hour's time] ...+00:00"
#     }
# }
aws iam list-policies
# {
#     "Policies": [
#         {
#             "PolicyName": "AdministratorAccess",
#             "PolicyId": "ANP...
# ...
```

Press `q` to exit the `aws iam list-policies` command.

## Step 1.4: Create an S3 (Simple Storage Service) bucket

> __NOTE:__ The region must be specified as `us-east-1`, not your default region.

```bash
aws s3 ls
# (nothing listed)
aws s3api create-bucket --region us-east-1 --bucket tryout-auth-static-site-s3-bucket
# {
#     "Location": "/tryout-auth-static-site-s3-bucket"
# }
aws s3 ls
# 2025-03-04 ... tryout-auth-static-site-s3-bucket
```

## Step 1.5: Upload your static site to the S3 bucket

```bash
aws s3 ls s3://tryout-auth-static-site-s3-bucket/
# (nothing listed)
aws s3 sync --dryrun docs/ s3://tryout-auth-static-site-s3-bucket/
# (dryrun) upload: docs/index.html to s3://tryout-auth-static-site-s3-bucket/index.html
aws s3 sync docs/ s3://tryout-auth-static-site-s3-bucket/
# upload: docs/index.html to s3://tryout-auth-static-site-s3-bucket/index.html
aws s3 ls s3://tryout-auth-static-site-s3-bucket/
# 2025-03-04 ...        892 index.html
aws s3 rm --recursive --dryrun s3://tryout-auth-static-site-s3-bucket/
# (dryrun) delete: s3://tryout-auth-static-site-s3-bucket/index.html
aws s3 rm --recursive s3://tryout-auth-static-site-s3-bucket/
# delete: s3://tryout-auth-static-site-s3-bucket/index.html
aws s3 ls s3://tryout-auth-static-site-s3-bucket/
# (nothing listed)
aws s3 sync docs/ s3://tryout-auth-static-site-s3-bucket/
# upload: docs/index.html to s3://tryout-auth-static-site-s3-bucket/index.html
```

## Step 1.6: Make the S3 bucket a static website

```bash
aws s3 website s3://tryout-auth-static-site-s3-bucket/ --index-document index.html --error-document index.html
```

Visit <https://us-east-1.console.aws.amazon.com/s3/>, and in the ‘General
purpose buckets’ list click the ‘tryout-auth-static-site-s3-bucket’ link.

You should see the index.html’ file listed under ‘Objects’.

Click the ‘Properties’ tab, scroll to the ‘Static website hosting’ section, and
click <http://tryout-auth-static-site-s3-bucket.s3-website-us-east-1.amazonaws.com>.
You should see “403 Forbidden ... Access Denied”, because the bucket is private.

Scroll back up and click the ‘Permissions’ tab, and click ‘Edit’ in the
‘Block public access (bucket settings)’ section. Uncheck the ‘Block all public
access’ checkbox, and click the ‘Save changes’ button. Type “confirm” in the
confirmation dialog, and click the ‘Confirm’ button.

Scroll down to the ‘Bucket policy’ section, click the ‘Edit’ button, and paste
the following policy:

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "PublicReadGetObject",
			"Effect": "Allow",
			"Principal": "*",
			"Action": "s3:GetObject",
			"Resource": "arn:aws:s3:::tryout-auth-static-site-s3-bucket/*"
		}
	]
}
```

Click the ‘Save changes’ button.

Refresh the <http://tryout-auth-static-site-s3-bucket.s3-website-us-east-1.amazonaws.com>
page - you should see the “tryout-auth-static-site ...” page. Note that all
routes point to the index.html page, because we set `--error-document index.html`.
