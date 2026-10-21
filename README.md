# GuardDuty to Route 53 Resolver DNS Firewall (CloudFormation)

CloudFormation template that automatically adds domains from GuardDuty DNS findings (received through
Security Hub) to Route 53 Resolver DNS Firewall domain lists, blocking HIGH/CRITICAL ones and alerting on the rest.
A "remastered" version of the AWS sample from the blog post
[Automatically block suspicious DNS activity with Amazon GuardDuty and Route 53 Resolver DNS Firewall](https://aws.amazon.com/blogs/security/automatically-block-suspicious-dns-activity-with-amazon-guardduty-and-route-53-resolver-dns-firewall/).

> **Status:** legacy reference template (2026), not actively maintained. Review and update before deploying.

## Architecture

```
GuardDuty finding -> Security Hub -> EventBridge rule -> Step Functions state machine
   -> Lambda: domain already in DynamoDB?  -> Lambda: add to Block (HIGH/CRITICAL) or Alert list
   -> Lambda: record finding ID in DynamoDB -> SNS email (success / failure)
```

- **DNS Firewall**: two domain lists (block, alert), a rule group (BLOCK with `NODATA`/`NXDOMAIN`,
  then ALERT) associated with one VPC.
- **Resolver query logging** for the VPC to a CloudWatch log group (`dnsFirewallLogGroup-<VpcId>`, 7 days).
- **DynamoDB table** (on-demand, key `DomainName`) storing the GuardDuty finding IDs per domain.
- **Three Lambda functions** (Python 3.8, inline code) and their IAM roles.
- **Step Functions state machine** orchestrating the lambdas, with retries and SNS notifications.
- **EventBridge rule** matching Security Hub findings that contain the GuardDuty
  `dnsRequestAction` field, plus its IAM role.
- **SNS topic** with an email subscription for administrators.

`sample_guardutty_DNS_Attack.txt` is a sample Security Hub/GuardDuty event (example account ID, C&C
domain finding) you can use to test the state machine.

## Parameters

| Name | Purpose | Default |
|------|---------|---------|
| `AdminEmail` | Email subscribed to the SNS notifications | `<EMAIL>` |
| `DnsFireWallBlockDomainListName` | Name of the block domain list | `BlockedDomainListAutoUpdated` |
| `DnsFirewallBlockAction` | Block response: `NODATA` or `NXDOMAIN` | `NODATA` |
| `DnsFireWallAlertDomainListName` | Name of the alert domain list | `AlertDomainListAutoUpdated` |
| `VpcId` | VPC to protect and log | - |

## Deploy

Prerequisites: GuardDuty and Security Hub enabled in the region, with the GuardDuty integration active.

```bash
aws cloudformation deploy \
  --stack-name dns-firewall-guardduty \
  --template-file dnsFirewallCFTemplate-Remastered.yml \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides AdminEmail=<EMAIL> VpcId=<VPC_ID>
```

Confirm the SNS subscription email. To test, start an execution of the state machine with the content of
`sample_guardutty_DNS_Attack.txt` as input.

## Security notes

- The lambda runtime `python3.8` is deprecated; update it before deploying.
- The DNS Firewall Lambda runs `pip install boto3` into `/tmp` on every cold start: it needs internet
  access and pulls an unpinned package at runtime. Use the runtime's boto3 or a layer instead.
- The Lambda role can update **any** DNS Firewall domain list (`Resource: "*"`).
- Domains are added automatically with no review step, so a false positive is blocked VPC-wide.
- The rule group association has `MutationProtection: DISABLED`.
- DynamoDB table, SNS topic and log group use default encryption (no customer-managed KMS key).

## Cleanup

```bash
aws cloudformation delete-stack --stack-name dns-firewall-guardduty
```

The Lambda functions' own CloudWatch log groups (`/aws/lambda/...`) are created at runtime and must be deleted manually.

## License

MIT — see [LICENSE](LICENSE). The template header carries the original AWS sample's MIT No Attribution notice.
