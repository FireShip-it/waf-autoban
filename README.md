# AWS WAF Auto-Ban: Score-Based IP Blocking with Lambda and DynamoDB

Automatically ban IP addresses that repeatedly trigger AWS WAF block rules using a scoring system with configurable thresholds and ban expiry.

## Why this solution?

AWS WAF rate-based rules unblock IPs automatically when their request count drops below the threshold. This solution provides:

- **Persistent bans** — IPs stay blocked for a configurable duration regardless of request rate
- **Score-based decisions** — accumulate a score over time before banning, reducing false positives
- **Label awareness** — score differently based on which WAF rule triggered the block
- **Automatic expiry** — DynamoDB TTL removes expired bans without additional logic

## Architecture

```
CloudWatch Logs (WAF) 
    │
    ▼ (subscription filter: BLOCK events only)
Lambda (Scorer)
    │
    ▼ (increment per-IP score)
DynamoDB (IP scores + ban expiry)
    │
    ▼ (scheduled read)
EventBridge (every 1 minute)
    │
    ▼
Lambda (Enforcer)
    │
    ▼ (update IP Set)
AWS WAF (IP Set rule blocks banned IPs)
```

## Components

| Component | Purpose |
|---|---|
| WAF Web ACL | Logs BLOCK events to CloudWatch Logs |
| CloudWatch Logs subscription filter | Forwards BLOCK events to the scorer Lambda |
| Scorer Lambda | Parses events, increments per-IP score in DynamoDB |
| DynamoDB table | Stores IP address, score, and ban expiry timestamp |
| EventBridge rule | Triggers enforcer Lambda every minute |
| Enforcer Lambda | Reads DynamoDB, updates WAF IP Set with banned IPs |
| WAF IP Set | Contains currently banned IPs |
| WAF IP Set rule | Blocks requests from IPs in the ban list |

## Configuration

| Parameter | Default | Description |
|---|---|---|
| `BanThreshold` | 10 | Score required to trigger a ban |
| `BanDurationMinutes` | 60 | How long an IP stays banned |
| `ScorePerBlock` | 1 | Points added per BLOCK event |
| `ScoreDecayMinutes` | 5 | Time window for score accumulation |

## Deployment

### Prerequisites

- AWS CLI configured with appropriate permissions
- An existing WAF Web ACL with logging enabled to CloudWatch Logs

### Deploy the stack

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name waf-autoban \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
    WebAclArn=arn:aws:wafv2:us-east-1:123456789012:regional/webacl/my-web-acl/abc123 \
    LogGroupName=aws-waf-logs-my-web-acl
```

### Connect the subscription filter

After deployment, create the subscription filter:

```bash
aws logs put-subscription-filter \
  --log-group-name aws-waf-logs-my-web-acl \
  --filter-name waf-autoban-scorer \
  --filter-pattern '{ $.action = "BLOCK" }' \
  --destination-arn arn:aws:lambda:us-east-1:123456789012:function:waf-autoban-scorer
```

## Cost estimate

For a workload with 10,000 BLOCK events per day:

| Service | Monthly cost |
|---|---|
| DynamoDB (on-demand) | ~$0.50 |
| Lambda (scorer + enforcer) | ~$0.10 |
| CloudWatch Logs (subscription filter) | Included in existing WAF logging |
| EventBridge | Free tier |
| **Total** | **~$0.60/month** |

## How it works

1. **WAF blocks a request** and logs the event to CloudWatch Logs
2. **Subscription filter** matches BLOCK events and invokes the scorer Lambda
3. **Scorer Lambda** extracts the source IP and increments its score in DynamoDB
4. **If the score exceeds the threshold**, the scorer sets a ban expiry timestamp
5. **Enforcer Lambda** runs every minute, reads all IPs with active bans, and updates the WAF IP Set
6. **WAF IP Set rule** blocks all requests from IPs in the ban list
7. **DynamoDB TTL** automatically deletes expired ban records

## Customization

### Score by label

Modify the scorer Lambda to assign different scores based on WAF labels:

```python
label_scores = {
    'awswaf:managed:aws:sql-database': 5,
    'awswaf:managed:aws:linux-os': 3,
    'awswaf:managed:aws:core-rule-set': 2,
}

for label in event.get('labels', []):
    if label['name'] in label_scores:
        score += label_scores[label['name']]
```

### Allowlist

Add an allowlist check in the enforcer Lambda:

```python
allowlist = ['203.0.113.0/24', '198.51.100.0/24']

def is_allowed(ip):
    for cidr in allowlist:
        if ip_in_cidr(ip, cidr):
            return True
    return False
```

## Cleanup

```bash
aws cloudformation delete-stack --stack-name waf-autoban
```

## License

MIT

## References

- [AWS WAF logging](https://docs.aws.amazon.com/waf/latest/developerguide/logging.html)
- [DynamoDB TTL](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html)
- [CloudWatch Logs subscription filters](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/SubscriptionFilters.html)
