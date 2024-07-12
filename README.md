# Cross-Account Observability - CloudWatch Alarms Replicator

![diagram](/images/solution_diagram.png)

### Description
This proof of concept solution allow you to replicate alarms creation/update/delete operations from Source accounts into a Monitoring account leveraging Amazon EventBridge and AWS Lambda. Based on cross-account EventBridge events of type "CloudWatch Alarm Configuration Change".

### Prerequisites/Assumptions
- [CloudWatch cross-account observability](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Unified-Cross-Account.html) is already in place. With a "Monitoring" account configured as such and connected with "Source" accounts in your preferred AWS Region(s).

### Considerations
- This solution supports the replication of Alarms tracking Metric Insights queries, Anomaly Detection metrics, single metrics and math expressions. It also support the replication of Composite Alarms but, only for those composed of alarms already replicated into the "Monitoring" account.
- Alarms are created in the "Monitoring" account following a naming convention as “[<SOURCE_ACCOUNT_ID>]-<ALARM_NAME>”
- Alarms’ related actions are not replicated.
- Consider increasing the TPS quotas of the CloudWatch PutMetricAlarm and PutCompositeAlarm operations (to avoid throttling errors in the "Monitoring" account). For DeleteAlarms operation, currently, limits cannot be increased.
- It is recommended to have a separate logic/solution to rebase the alarms in the monitoring account (something that can be ran before implementing this solution to ensure parity)

### Cost
- There is a cost associated for each CloudWatch Alarms created in an account. This solution will create a copy of every new CloudWatch Alarms from a "Source" account into the "Monitoring" account.For this, there will be a cost of each new alarm copied in the "Monitoring" account.
- Refer to the CloudWatch Alarms pricing documentation for more information: https://aws.amazon.com/cloudwatch/pricing/

## Setup Instructions

### Step 1 - In Monitoring Account 
1. Open the CloudFormation console in the AWS Region where CloudWatch cross-account observability have been previously setup.
2. Deploy the template [MonitoringAccount_CrossAccountCloudWatchAlarmReplicator.yml](templates/MonitoringAccount_CrossAccountCloudWatchAlarmReplicator.yml)
3. For the template parameters:
   1. *AccountIds*: The comma-separated list of Account IDs. Add this to allow this list of specific accounts to send "create", "update" and "delete" alarms events to the monitoring account. (Optional - Either OrganizationId or AccountIds parameters or both should be included)
   2. *OrganizationId*: The Organization ID. Add this to allow any account within the organization to send "create", "update" and "delete" alarms events to the monitoring account. (Optional - Either OrganizationId or AccountIds parameters or both should be included)
   3. *EventBusName*: The name for the custom EventBridge Bus that will be receiving the CloudWatch Alarm "create", "update" and "delete" events.
   4. *LambdaFunctionName*: The name for the Lambda function creating a copy of CloudWatch alarms from source account in the monitoring account.
4. Copy the value of **EventBusArn** from the stack Outputs.

### Step 2 - In Source Account(s)
1. Open the CloudFormation console in the same AWS Region as in "Step 1" above.
2. Deploy the template [SourceAccount_CrossAccountCloudWatchAlarmReplicator.yml](templates/SourceAccount_CrossAccountCloudWatchAlarmReplicator.yml)
3. For the template parameters:
   1. *MonitoringAccountEventBusArn*: Add the value of the **EventBusArn** from "Step 1" above.
