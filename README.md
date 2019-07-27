# Video Encoding Pipeline

This repository is a simple walk through of the solution described in
https://aws.amazon.com/solutions/video-on-demand-on-aws/

# Configure the CloudFormation Stack

Edit the `parameters.json` file and replace `<YOUR_EMAIL_HERE>` with your email address

AWS SNS will use this email address to send you emails reporting the status of the encoding workflows

# Deploy the CloudFormation Stack

Run

```sh
aws cloudformation create-stack \
    --stack-name $USER \
    --template-url https://s3.amazonaws.com/solutions-reference/video-on-demand-on-aws/latest/video-on-demand-on-aws.template \
    --parameters file:///$PWD/parameters.json \
    --capabilities CAPABILITY_IAM
```

You will see output like:

```json
{
    "StackId": "arn:aws:cloudformation:us-east-1:881926258290:stack/ttruma200/d3333170-b084-11e9-bc53-12a3b4ca1ea2"
}
```

# Confirm the Email Subscription

Go to your inbox and look for an email with the subject line: `AWS Notification - Subscription Confirmation`

Open the email and click the link `Confirm subscription`

# Check the Stack Status

Run `aws cloudformation describe-stack-events --stack-name $USER`

You should see a bunch of output that looks like:

```json
{
    "StackEvents": [
        {
            "StackId": "arn:aws:cloudformation:us-east-1:881926258290:stack/ttruma200/d3333170-b084-11e9-bc53-12a3b4ca1ea2",
            "EventId": "CloudWatchLambdaInvokeCompletes-CREATE_COMPLETE-2019-07-27T15:50:52.315Z",
            "ResourceStatus": "CREATE_COMPLETE",
            "ResourceType": "AWS::Lambda::Permission",
            "Timestamp": "2019-07-27T15:50:52.315Z",
            "StackName": "ttruma200",
            "ResourceProperties": "{\"FunctionName\":\"arn:aws:lambda:us-east-1:881926258290:function:ttruma200-step-functions\",\"Action\":\"lambda:InvokeFunction\",\"SourceArn\":\"arn:aws:events:us-east-1:881926258290:rule/ttruma200-EncodeComplete\",\"Principal\":\"events.amazonaws.com\"}",
            "PhysicalResourceId": "ttruma200-CloudWatchLambdaInvokeCompletes-6M4XLK5LXRMV",
            "LogicalResourceId": "CloudWatchLambdaInvokeCompletes"
        },
        .....
}
```

# Find the S3 Buckets created by the Stack

Run `aws s3api list-buckets`

You should see buckets listed like:

```json
{
    "Owner": {
        "DisplayName": "info",
        "ID": "98a3b7153ff2444a9b5fb5b3cd33c0c08f4ba1fa1e4a329011d74c3b4911f704"
    },
    "Buckets": [
        {
            "CreationDate": "2019-07-27T15:48:32.000Z",
            "Name": "ttruma200-destination-yhislrv37ttj"
        },
        {
            "CreationDate": "2019-07-27T15:48:09.000Z",
            "Name": "ttruma200-logs-45qbdiu17ore"
        },
        {
            "CreationDate": "2019-07-27T15:48:09.000Z",
            "Name": "ttruma200-source-exgsovx1230f"
        },
        ......
    ]
}
```

# Upload Some Videos!

Find the bucket named like `$USER-source-....` in the output from `aws s3api list-buckets`

Run `aws s3 cp . s3://ttruma200-source-exgsovx1230f/ --recursive --exclude "*" --include "*.mp4"`

# Check the Encoding Workflow Status

Open your email inbox and you will see a bunch of emails with subjects starting with `Workflow Status:: `

Each email will provide you information about the encoding status of each video you uploaded.

# View the various encodings

If you open any of the emails with a subject starting with `Workflow Status:: Complete:: ` you can
find the CloudFront URLs for the various output formats. You should see URLs for:

 * mp4 at various resolutions - Can be downloaded directly and played with QuickTime player
 * [HLS](https://en.wikipedia.org/wiki/HTTP_Live_Streaming) for streaming to various players/devices
 * [DASH](https://en.wikipedia.org/wiki/Dynamic_Adaptive_Streaming_over_HTTP) for streaming to various players/devices

To open a streaming video on your mobile device, access your email from your mobile device and open one of the
URLs that look like `https://d2ixjb3a7th1zy.cloudfront.net/d6dee192-c7f5-4a71-a586-26906ef0dd02/hls/flowers.m3u8`

# View the S3 output produced by the encoding workflow

Find the bucket named like `$USER-destination-....` in the output from `aws s3api list-buckets`

Run `aws s3 ls s3://ttruma200-destination-yhislrv37ttj`

You should see output like:

```sh
PRE 170076e4-d044-49e5-b118-d4c0396fe382/
PRE d6dee192-c7f5-4a71-a586-26906ef0dd02/
```

These are S3 object prefixes, you can think of them like directories in a filesystem.

Let's list the contents of one of these prefixes:

`aws s3 ls s3://ttruma200-destination-yhislrv37ttj/170076e4-d044-49e5-b118-d4c0396fe382/`

You should see output like:

```sh
PRE dash/
PRE hls/
PRE mp4/
```

Let's list the contents of the mp4 prefix:

`aws s3 ls s3://ttruma200-destination-yhislrv37ttj/170076e4-d044-49e5-b118-d4c0396fe382/mp4/`

And here, you will see output like:

```sh
2019-07-27 12:11:01   21882183 sea_Mp4_Avc_Aac_16x9_1280x720p_24Hz_4.5Mbps_qvbr.mp4
2019-07-27 12:11:01   36134488 sea_Mp4_Avc_Aac_16x9_1920x1080p_24Hz_6Mbps_qvbr.mp4
2019-07-27 12:11:11  126830719 sea_Mp4_Hevc_Aac_16x9_3840x2160p_24Hz_20Mbps_qvbr.mp4
```

Which shows that our source MP4 video was encoded into 3 different resolutions.

# CloudFront distributions of our video content

Run `aws cloudfront list-distributions` which will produce output like:

```json
{
    "DistributionList": {
        "Items": [
            {
                "Status": "Deployed",
                "CacheBehaviors": {
                    "Quantity": 0
                },
                "Restrictions": {
                    "GeoRestriction": {
                        "RestrictionType": "none",
                        "Quantity": 0
                    }
                },
                "Origins": {
                    "Items": [
                        {
                            "S3OriginConfig": {
                                "OriginAccessIdentity": "origin-access-identity/cloudfront/E2UUDKDQQHMTJG"
                            },
                            "OriginPath": "",
                            "CustomHeaders": {
                                "Quantity": 0
                            },
                            "Id": "vodS3Origin",
                            "DomainName": "ttruma200-destination-yhislrv37ttj.s3.us-east-1.amazonaws.com"
                        }
                    ],
                    "Quantity": 1
                },
                "DomainName": "d2ixjb3a7th1zy.cloudfront.net",
                "WebACLId": "",
                "PriceClass": "PriceClass_100",
                "Enabled": true,
                "DefaultCacheBehavior": {
                    "FieldLevelEncryptionId": "",
                    "TrustedSigners": {
                        "Enabled": false,
                        "Quantity": 0
                    },
                    "LambdaFunctionAssociations": {
                        "Quantity": 0
                    },
                    "TargetOriginId": "vodS3Origin",
                    "ViewerProtocolPolicy": "allow-all",
                    "ForwardedValues": {
                        "Headers": {
                            "Items": [
                                "Access-Control-Request-Headers",
                                "Access-Control-Request-Method",
                                "Origin"
                            ],
                            "Quantity": 3
                        },
                        "Cookies": {
                            "Forward": "none"
                        },
                        "QueryStringCacheKeys": {
                            "Quantity": 0
                        },
                        "QueryString": false
                    },
                    "MaxTTL": 31536000,
                    "SmoothStreaming": false,
                    "DefaultTTL": 86400,
                    "AllowedMethods": {
                        "Items": [
                            "HEAD",
                            "GET",
                            "OPTIONS"
                        ],
                        "CachedMethods": {
                            "Items": [
                                "HEAD",
                                "GET"
                            ],
                            "Quantity": 2
                        },
                        "Quantity": 3
                    },
                    "MinTTL": 0,
                    "Compress": false
                },
                "IsIPV6Enabled": true,
                "Comment": "",
                "HttpVersion": "HTTP1_1",
                "ViewerCertificate": {
                    "CloudFrontDefaultCertificate": true,
                    "MinimumProtocolVersion": "TLSv1",
                    "CertificateSource": "cloudfront"
                },
                "CustomErrorResponses": {
                    "Quantity": 0
                },
                "LastModifiedTime": "2019-07-27T15:49:02.118Z",
                "OriginGroups": {
                    "Quantity": 0
                },
                "Id": "E16VI0Z3P5GAZY",
                "ARN": "arn:aws:cloudfront::881926258290:distribution/E16VI0Z3P5GAZY",
                "Aliases": {
                    "Quantity": 0
                }
            }
        ]
    }
}
```

This shows that our CloudFormation Stack set-up a CloudFront distribution of the videos
in our S3 bucket. This CloudFront distribution will give our user's faster access to our videos
by utilizing edge caching.

# Delete the Stack

Once you are done exploring the pipeline, you should delete the stack by running: `aws cloudformation delete-stack $USER`

Deletion can take 5-10 minutes.