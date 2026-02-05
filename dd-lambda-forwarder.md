# Datadog Integration Lambda - S3 Log Forwarder

## Overview
This document describes the setup and configuration of the Datadog Lambda forwarder (`dd-afulogs-test`) that automatically forwards S3 logs to Datadog for monitoring and analysis.

## Architecture

```
S3 Bucket (afu-access-logs--processed)
    └─> Event Notification (ObjectCreated)
        └─> Lambda Function (dd-afulogs-test)
            └─> Datadog API
```

## Configuration Details

### Lambda Function
- **Name**: `dd-afulogs-test`
- **Runtime**: Python 3.14
- **Region**: us-west-2
- **Account**: 405675796529
- **Deployment Method**: CloudFormation (Datadog official template)

### S3 Trigger Configuration
- **Bucket**: `afu-access-logs--processed`
- **Event Type**: `s3:ObjectCreated:*`
- **Prefix Filter**: `split/h19005_www1_hp_com/`
- **Event Notification ID**: `dd-afulogs-test-trigger`

### Secrets Management
- **Secret Name**: `dd-api-afu-logs`
- **Secret ARN**: `arn:aws:secretsmanager:us-west-2:405675796529:secret:dd-api-afu-logs-vZIJi4`
- **Secret Format**: Plain text API key (not JSON)
  ```
  19385a1ec63c2d617c56c5304b0d266b
  ```

### Datadog Configuration
- **Datadog Site**: datadoghq.com
- **API Key Source**: AWS Secrets Manager
- **Environment**: tools-dev

## Setup Steps

### 1. Create Secret in AWS Secrets Manager
```powershell
aws secretsmanager create-secret \
  --name dd-api-afu-logs \
  --secret-string "YOUR_DATADOG_API_KEY" \
  --region us-west-2
```

**Important**: The secret must contain **only the API key string**, not a JSON object. The Datadog forwarder expects plain text format.

### 2. Deploy Lambda via CloudFormation
```powershell
aws cloudformation create-stack \
  --stack-name dd-afulogs-test \
  --template-url https://datadog-cloudformation-template.s3.amazonaws.com/aws/forwarder/latest.yaml \
  --parameters \
    ParameterKey=DdApiKeySecretArn,ParameterValue=arn:aws:secretsmanager:us-west-2:405675796529:secret:dd-api-afu-logs-vZIJi4 \
    ParameterKey=DdSite,ParameterValue=datadoghq.com \
    ParameterKey=FunctionName,ParameterValue=dd-afulogs-test \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND \
  --region us-west-2
```

### 3. Grant S3 Invoke Permission
```powershell
aws lambda add-permission \
  --function-name dd-afulogs-test \
  --statement-id s3-invoke-permission \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::afu-access-logs--processed \
  --region us-west-2
```

### 4. Configure S3 Event Notification
Create `s3-notification.json`:
```json
{
  "LambdaFunctionConfigurations": [
    {
      "Id": "dd-afulogs-test-trigger",
      "LambdaFunctionArn": "arn:aws:lambda:us-west-2:405675796529:function:dd-afulogs-test",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": {
        "Key": {
          "FilterRules": [
            {
              "Name": "prefix",
              "Value": "split/h19005_www1_hp_com/"
            }
          ]
        }
      }
    }
  ]
}
```

Apply configuration:
```powershell
aws s3api put-bucket-notification-configuration \
  --bucket afu-access-logs--processed \
  --notification-configuration file://s3-notification.json \
  --region us-west-2
```

## Issues Encountered and Solutions

### Issue 1: Missing ujson Module
**Error**:
```
Runtime.ImportModuleError: Unable to import module 'lambda_function': No module named 'ujson'
```

**Root Cause**: Manually uploaded ZIP file from GitHub releases page contained source code only, not the compiled deployment package with all Python dependencies.

**Solution**: 
1. Download the correct deployment package: `aws-dd-forwarder-5.2.2.zip` (NOT "Source code.zip")
2. Or use CloudFormation deployment (recommended) which automatically includes all dependencies

**Prevention**: Always use CloudFormation template for Datadog Forwarder deployment to ensure proper dependency packaging.

---

### Issue 2: Cross-Account Secret Access Denied
**Error**:
```
AccessDeniedException: User is not authorized to perform: secretsmanager:GetSecretValue on resource: arn:aws:secretsmanager:us-west-2:281598178188:secret:datadog-api-key
```

**Root Cause**: Lambda was deployed in account `405675796529` but configured to access secret in account `281598178188`.

**Solution**: 
- Created new secret in the same AWS account as the Lambda function
- Updated CloudFormation stack parameter `DdApiKeySecretArn` to point to correct account secret

**Prevention**: Ensure all resources (Lambda, Secrets Manager) are in the same AWS account unless cross-account access is explicitly configured.

---

### Issue 3: Invalid Secret Format
**Error**:
```
Missing Datadog API key. Set DD_API_KEY environment variable.
```

**Root Cause**: Secret stored as JSON object `{"tools-dev":"API_KEY"}` but Datadog Forwarder expects plain text API key string.

**Solution**:
```powershell
aws secretsmanager update-secret \
  --secret-id dd-api-afu-logs \
  --secret-string "19385a1ec63c2d617c56c5304b0d266b" \
  --region us-west-2
```

**Prevention**: When creating Datadog API key secrets, store as **plain text string** only, not JSON.

**Correct Format**:
```
YOUR_API_KEY_HERE
```

**Incorrect Format** (Do NOT use):
```json
{"api_key": "YOUR_API_KEY_HERE"}
```

---

### Issue 4: S3 Event Notification vs Lambda Trigger Confusion
**Question**: "Why do we need event notification when we have trigger in function?"

**Answer**: They work together but serve different purposes:
- **S3 Event Notification** (S3 side): Actual configuration that sends events to Lambda
- **Lambda Trigger** (Lambda console): Display-only view showing which services can invoke the Lambda

The S3 event notification is the **real configuration**. The Lambda trigger is just a **visual representation** in the Lambda console. You cannot create the trigger from the Lambda side alone - it must be configured on the S3 bucket.

---

## Verification

### Check Lambda Execution
```powershell
# Upload test file
echo "test log" > test.log
aws s3 cp test.log s3://afu-access-logs--processed/split/h19005_www1_hp_com/test.log --region us-west-2

# Check CloudWatch Logs (if AWS CLI v2 with tail support)
aws logs tail /aws/lambda/dd-afulogs-test --follow --region us-west-2
```

### Check S3 Event Configuration
**AWS Console**: 
1. Go to S3 console → `afu-access-logs--processed` bucket
2. Click **Properties** tab
3. Scroll to **Event notifications** section
4. Verify `dd-afulogs-test-trigger` is listed

**CLI**:
```powershell
aws s3api get-bucket-notification-configuration \
  --bucket afu-access-logs--processed \
  --region us-west-2
```

### Check Datadog Logs
1. Log in to Datadog: https://app.datadoghq.com
2. Navigate to **Logs** section
3. Filter by source: `s3` or service: `dd-afulogs-test`
4. Verify logs are appearing from S3 bucket

## Maintenance

### Disable Log Forwarding
To temporarily stop forwarding logs without deleting the Lambda:
```powershell
aws s3api put-bucket-notification-configuration \
  --bucket afu-access-logs--processed \
  --notification-configuration '{}' \
  --region us-west-2
```

### Re-enable Log Forwarding
Reapply the S3 notification configuration using the JSON file from step 4 of setup.

### Update Datadog API Key
```powershell
aws secretsmanager update-secret \
  --secret-id dd-api-afu-logs \
  --secret-string "NEW_API_KEY" \
  --region us-west-2
```

Note: Lambda will automatically pick up the new secret on next invocation (no restart needed).

### Delete Stack
```powershell
aws cloudformation delete-stack \
  --stack-name dd-afulogs-test \
  --region us-west-2
```

## Troubleshooting

### Logs Not Appearing in Datadog
1. **Check Lambda execution**: View CloudWatch Logs for errors
2. **Verify API key**: Ensure secret contains valid Datadog API key
3. **Check S3 permissions**: Lambda role must have `s3:GetObject` permission on bucket
4. **Verify network**: Lambda must have internet access (via NAT Gateway or VPC endpoints)
5. **Check Datadog site**: Ensure `DdSite` parameter matches your Datadog account region

### High Lambda Costs
- Review S3 prefix filter to avoid processing unnecessary files
- Consider adding file extension filter (e.g., only `.log` files)
- Enable S3 Lifecycle policies to archive old logs

### Lambda Timeout Errors
- Default timeout: 120 seconds
- For large log files, increase timeout in CloudFormation parameters:
  ```
  ParameterKey=FunctionTimeout,ParameterValue=300
  ```

## Best Practices

1. **Use CloudFormation**: Always deploy Datadog Forwarder via official CloudFormation template
2. **Plain Text Secrets**: Store Datadog API keys as plain text in Secrets Manager
3. **Same Account Resources**: Keep Lambda and secrets in the same AWS account
4. **Prefix Filtering**: Use specific S3 prefix filters to avoid processing unwanted files
5. **Monitor Costs**: Set up billing alerts for Lambda invocations and data transfer
6. **Log Retention**: Configure CloudWatch Logs retention policy (default: never expire)
7. **Stack Updates**: Use `aws cloudformation update-stack` to modify configuration, don't edit Lambda directly

## References

- [Datadog Forwarder Documentation](https://docs.datadoghq.com/logs/guide/forwarder/)
- [Datadog CloudFormation Template](https://github.com/DataDog/datadog-serverless-functions/tree/master/aws/logs_monitoring)
- [S3 Event Notifications](https://docs.aws.amazon.com/AmazonS3/latest/userguide/NotificationHowTo.html)
- [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/)

## Change Log

| Date | Version | Author | Changes |
|------|---------|--------|---------|
| 2026-02-04 | 1.0 | Sai Ravilla | Initial setup and documentation |

---

**Last Updated**: February 4, 2026  
**Maintained By**: DevOps Team  
**Support Contact**: infrastructure-team@company.com
