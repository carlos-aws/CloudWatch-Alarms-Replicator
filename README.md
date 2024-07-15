# Cross-Account Observability - CloudWatch Alarms Replicator

![diagram](/images/solution_diagram.png)

### Description
This proof of concept solution allows you to replicate alarms create/update/delete operations from Source accounts into a Monitoring account leveraging Amazon EventBridge and AWS Lambda services.

**Workflow**
1. A CloudWatch Alarm is created, updated, or deleted in a "Source" account.
2. An EventBridge rule in the "Source" account tracks the create/update/delete operation as a CloudWatch Alarm event of type ["CloudWatch Alarm Configuration Change"](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch-and-eventbridge.html). The event is forwarded cross-account to a custom event bus in the "Monitoring" account.
3. An EventBridge rule in the "Monitoring" account tracks the operation received by the custom event bus and invokes the Lambda function in the "Monitoring" account.
4. The Lambda function replicates the same create/update/delete operation in the "Monitoring" account.

### Prerequisites/Assumptions
- [CloudWatch cross-account observability](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Unified-Cross-Account.html) should be already in place. With a "Monitoring" account configured as such and connected with "Source" accounts in your preferred AWS Region(s).

### Considerations
- This solution supports the replication of CloudWatch Alarms tracking [Metric Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch-metrics-insights-alarm-create.html) queries, [Anomaly Detection](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Create_Anomaly_Detection_Alarm.html) metrics, [single metrics](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ConsoleAlarms.html), and [math expressions](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Create-alarm-on-metric-math-expression.html). It also supports the replication of [Composite Alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Create_Composite_Alarm.html), but only for those composed of alarms already replicated into the "Monitoring" account.
- Alarms are created in the "Monitoring" account following a naming convention as **[<SOURCE_ACCOUNT_ID>]-<ALARM_NAME>**.
- Any [actions](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html#alarms-and-actions) that an alarm has configured in a "Source" account is not included when creating or updating the alarm replica in the "Monitoring" account. This means that [Alarms actions](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html#alarms-and-actions) will need to be independently managed and configured in the "Monitoring" account.
- In the "Monitoring" account, consider increasing the Transactions Per Second (TPS) [quotas of the CloudWatch PutMetricAlarm and PutCompositeAlarm operations](https://docs.aws.amazon.com/servicequotas/latest/userguide/configure-cloudwatch.html) (to avoid throttling errors). For the DeleteAlarms operation, limits cannot be increased currently.
- It is recommended to have a separate logic/solution to rebase the alarms in the "Monitoring" account (something that can be run before implementing this solution to ensure parity).

### Cost
- There is a cost associated with each CloudWatch Alarm created in an account. This solution will create a copy of every new CloudWatch Alarm from a "Source" account into the "Monitoring" account. For this, there will be a cost for each new alarm copied in the "Monitoring" account.
- Refer to the CloudWatch Alarms pricing documentation for more information: https://aws.amazon.com/cloudwatch/pricing/

## Setup Instructions

### Step 1 - In the Monitoring Account
1. Open the CloudFormation console in the AWS Region where CloudWatch cross-account observability has been previously set up.
2. Deploy the template [MonitoringAccount_CrossAccountCloudWatchAlarmReplicator.yml](templates/MonitoringAccount_CrossAccountCloudWatchAlarmReplicator.yml).
3. For the template parameters:
   1. *AccountIds*: The comma-separated list of Account IDs. Add this to allow this list of specific accounts to send "create", "update", and "delete" alarms events to the monitoring account. (Note - Add either the OrganizationId parameter or the AccountIds parameter for deploying the stack, not both.)
   2. *OrganizationId*: The Organization ID. Add this to allow any account within the organization to send "create", "update", and "delete" alarms events to the monitoring account. (Note - Add either the OrganizationId parameter or the AccountIds parameter for deploying the stack, not both.)
4. Copy the value of **EventBusArn** from the stack Outputs.

### Step 2 - In the Source Account(s)
1. Open the CloudFormation console in the same AWS Region as in "Step 1" above.
2. Deploy the template [SourceAccount_CrossAccountCloudWatchAlarmReplicator.yml](templates/SourceAccount_CrossAccountCloudWatchAlarmReplicator.yml).
3. For the template parameters:
   1. *MonitoringAccountEventBusArn*: Add the value of the **EventBusArn** from "Step 1" above.
4. Once this stack is successfully deployed, any new operation for creating, updating or deleting a CloudWatch Alarm in the "Source" account will be replicated in the "Monitoring" account.
