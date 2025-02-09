# Google SecOps SDK for Python

A Python SDK for interacting with Google Security Operations products, currently supporting Chronicle.

## Installation

```bash
pip install -i https://test.pypi.org/simple/ secops
```

## Authentication

The SDK supports two main authentication methods:

### 1. Application Default Credentials (ADC)

The simplest and recommended way to authenticate the SDK. Application Default Credentials provide a consistent authentication method that works across different Google Cloud environments and local development.

There are several ways to use ADC:

#### a. Using `gcloud` CLI (Recommended for Local Development)

```bash
# Login and set up application-default credentials
gcloud auth application-default login
```

Then in your code:
```python
from secops import SecOpsClient

# Initialize with default credentials - no explicit configuration needed
client = SecOpsClient()
```

#### b. Using Environment Variable

Set the environment variable pointing to your service account key:
```bash
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/service-account.json"
```

Then in your code:
```python
from secops import SecOpsClient

# Initialize with default credentials - will automatically use the credentials file
client = SecOpsClient()
```

#### c. Google Cloud Environment (Automatic)

When running on Google Cloud services (Compute Engine, Cloud Functions, Cloud Run, etc.), ADC works automatically without any configuration:

```python
from secops import SecOpsClient

# Initialize with default credentials - will automatically use the service account 
# assigned to your Google Cloud resource
client = SecOpsClient()
```

ADC will automatically try these authentication methods in order:
1. Environment variable `GOOGLE_APPLICATION_CREDENTIALS`
2. Google Cloud SDK credentials (set by `gcloud auth application-default login`)
3. Google Cloud-provided service account credentials
4. Local service account impersonation credentials

### 2. Service Account Authentication

For more explicit control, you can authenticate using a service account. This can be done in two ways:

#### a. Using a Service Account JSON File

```python
from secops import SecOpsClient

# Initialize with service account JSON file
client = SecOpsClient(service_account_path="/path/to/service-account.json")
```

#### b. Using Service Account Info Dictionary

```python
from secops import SecOpsClient

# Service account details as a dictionary
service_account_info = {
    "type": "service_account",
    "project_id": "your-project-id",
    "private_key_id": "key-id",
    "private_key": "-----BEGIN PRIVATE KEY-----\n...",
    "client_email": "service-account@project.iam.gserviceaccount.com",
    "client_id": "client-id",
    "auth_uri": "https://accounts.google.com/o/oauth2/auth",
    "token_uri": "https://oauth2.googleapis.com/token",
    "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
    "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/..."
}

# Initialize with service account info
client = SecOpsClient(service_account_info=service_account_info)
```

## Using the Chronicle API

### Initializing the Chronicle Client

After creating a SecOpsClient, you need to initialize the Chronicle-specific client:

```python
# Initialize Chronicle client
chronicle = client.chronicle(
    customer_id="your-chronicle-instance-id",  # Your Chronicle instance ID
    project_id="your-project-id",             # Your GCP project ID
    region="us"                               # Chronicle API region
)
```

### Basic UDM Search

Search for network connection events:

```python
from datetime import datetime, timedelta, timezone

# Set time range for queries
end_time = datetime.now(timezone.utc)
start_time = end_time - timedelta(hours=24)  # Last 24 hours

# Perform UDM search
results = chronicle.search_udm(
    query="""
    metadata.event_type = "NETWORK_CONNECTION"
    ip != ""
    """,
    start_time=start_time,
    end_time=end_time,
    max_events=5
)

# Example response:
{
    "events": [
        {
            "event": {
                "metadata": {
                    "eventTimestamp": "2024-02-09T10:30:00Z",
                    "eventType": "NETWORK_CONNECTION"
                },
                "target": {
                    "ip": "192.168.1.100",
                    "port": 443
                },
                "principal": {
                    "hostname": "workstation-1"
                }
            }
        }
    ],
    "total_events": 1
}
```

### Statistics Queries

Get statistics about network connections grouped by hostname:

```python
stats = chronicle.get_stats(
    query="""
    metadata.event_type = "NETWORK_CONNECTION"
    match:
        target.hostname
    outcome:
        $count = count(metadata.id)
    order:
        $count desc
    """,
    start_time=start_time,
    end_time=end_time,
    max_events=1000,
    max_values=10
)

# Example response:
{
    "columns": ["hostname", "count"],
    "rows": [
        {"hostname": "server-1", "count": 1500},
        {"hostname": "server-2", "count": 1200}
    ],
    "total_rows": 2
}
```

### CSV Export

Export specific fields to CSV format:

```python
csv_data = chronicle.fetch_udm_search_csv(
    query='metadata.event_type = "NETWORK_CONNECTION"',
    start_time=start_time,
    end_time=end_time,
    fields=[
        "metadata.eventTimestamp",
        "principal.hostname",
        "target.ip",
        "target.port"
    ]
)

# Example response:
"""
metadata.eventTimestamp,principal.hostname,target.ip,target.port
2024-02-09T10:30:00Z,workstation-1,192.168.1.100,443
2024-02-09T10:31:00Z,workstation-2,192.168.1.101,80
"""
```

### Query Validation

Validate a UDM query before execution:

```python
query = 'target.ip != "" and principal.hostname = "test-host"'
validation = chronicle.validate_query(query)

# Example response:
{
    "isValid": true,
    "queryType": "QUERY_TYPE_UDM_QUERY",
    "suggestedFields": [
        "target.ip",
        "principal.hostname"
    ]
}
```

### Entity Summary

Get detailed information about specific entities:

```python
# IP address summary
ip_summary = chronicle.summarize_entity(
    start_time=start_time,
    end_time=end_time,
    value="192.168.1.100"  # Automatically detects IP
)

# Domain summary
domain_summary = chronicle.summarize_entity(
    start_time=start_time,
    end_time=end_time,
    value="example.com"  # Automatically detects domain
)

# File hash summary
file_summary = chronicle.summarize_entity(
    start_time=start_time,
    end_time=end_time,
    value="e17dd4eef8b4978673791ef4672f4f6a"  # Automatically detects MD5
)

# Example response structure:
{
    "entities": [
        {
            "name": "entities/...",
            "metadata": {
                "entityType": "ASSET",
                "interval": {
                    "startTime": "2024-02-08T10:30:00Z",
                    "endTime": "2024-02-09T10:30:00Z"
                }
            },
            "metric": {
                "firstSeen": "2024-02-08T10:30:00Z",
                "lastSeen": "2024-02-09T10:30:00Z"
            },
            "entity": {
                "asset": {
                    "ip": ["192.168.1.100"]
                }
            }
        }
    ],
    "alertCounts": [
        {
            "rule": "Suspicious Network Connection",
            "count": 5
        }
    ],
    "widgetMetadata": {
        "detections": 5,
        "total": 1000
    }
}
```

### Entity Summary from Query

Look up entities based on a UDM query:

```python
# Search for a specific file hash across multiple UDM paths
md5 = "e17dd4eef8b4978673791ef4672f4f6a"
query = (
    f'principal.file.md5 = "{md5}" OR '
    f'principal.process.file.md5 = "{md5}" OR '
    f'target.file.md5 = "{md5}" OR '
    f'target.process.file.md5 = "{md5}" OR '
    f'security_result.about.file.md5 = "{md5}"'
)

results = chronicle.summarize_entities_from_query(
    query=query,
    start_time=start_time,
    end_time=end_time
)

# Example response:
[
    {
        "entities": [
            {
                "name": "entities/...",
                "metadata": {
                    "entityType": "FILE",
                    "interval": {
                        "startTime": "2024-02-08T10:30:00Z",
                        "endTime": "2024-02-09T10:30:00Z"
                    }
                },
                "metric": {
                    "firstSeen": "2024-02-08T10:30:00Z",
                    "lastSeen": "2024-02-09T10:30:00Z"
                },
                "entity": {
                    "file": {
                        "md5": "e17dd4eef8b4978673791ef4672f4f6a",
                        "sha1": "da39a3ee5e6b4b0d3255bfef95601890afd80709",
                        "filename": "suspicious.exe"
                    }
                }
            }
        ]
    }
]
```

### List IoCs (Indicators of Compromise)

You can retrieve IoC matches against ingested events:

```python
from datetime import datetime, timedelta, timezone
from secops import SecOpsClient

client = SecOpsClient()
chronicle = client.chronicle(
    customer_id="your-customer-id",
    project_id="your-project-id"
)

# Get IoCs from the last 24 hours
end_time = datetime.now(timezone.utc)
start_time = end_time - timedelta(hours=24)

iocs = chronicle.list_iocs(
    start_time=start_time,
    end_time=end_time,
    max_matches=1000,  # Maximum number of results to return
    add_mandiant_attributes=True,  # Include Mandiant attributes
    prioritized_only=False  # Include all IoCs, not just prioritized ones
)

# Process the results
for ioc in iocs['matches']:
    print(f"IoC Type: {next(iter(ioc['artifactIndicator'].keys()))}")
    print(f"IoC Value: {next(iter(ioc['artifactIndicator'].values()))}")
    print(f"Sources: {', '.join(ioc['sources'])}")
    print(f"Categories: {', '.join(ioc['categories'])}")
```

The response includes detailed information about each IoC match, including:
- The indicator itself (domain, IP, hash, etc.)
- Sources and categories
- Affected assets in your environment
- First and last seen timestamps
- Confidence scores and severity ratings
- Associated threat actors and malware families

## Error Handling

The SDK defines several custom exceptions:

```python
from secops.exceptions import SecOpsError, AuthenticationError, APIError

try:
    results = chronicle.search_udm(...)
except AuthenticationError as e:
    print(f"Authentication failed: {e}")
except APIError as e:
    print(f"API request failed: {e}")
except SecOpsError as e:
    print(f"General error: {e}")
```

## Value Type Detection

The SDK automatically detects the type of value being searched for entity summaries:

- IPv4 addresses
- MD5 hashes
- SHA1 hashes
- SHA256 hashes
- Domain names
- Email addresses
- MAC addresses
- Hostnames

Example of automatic detection:

```python
# These will automatically use the correct field paths and value types
ip_summary = chronicle.summarize_entity(value="192.168.1.100")
domain_summary = chronicle.summarize_entity(value="example.com")
hash_summary = chronicle.summarize_entity(value="e17dd4eef8b4978673791ef4672f4f6a")
```

You can also override the automatic detection by explicitly specifying `field_path` or `value_type`:

```python
summary = chronicle.summarize_entity(
    value="example.com",
    field_path="custom.field.path",  # Override automatic detection
    value_type="DOMAIN_NAME"         # Explicitly set value type
)
```

## Case Management

You can retrieve and manage Chronicle cases:

```python
from secops import SecOpsClient

client = SecOpsClient()
chronicle = client.chronicle(
    customer_id="your-customer-id",
    project_id="your-project-id"
)

# Get details for specific cases
cases = chronicle.get_cases(["case-id-1", "case-id-2"])

# Filter cases by priority
high_priority = cases.filter_by_priority("PRIORITY_HIGH")
for case in high_priority:
    print(f"High Priority Case: {case.display_name}")
    print(f"Stage: {case.stage}")
    print(f"Status: {case.status}")

# Look up a specific case
case = cases.get_case("case-id-1")
if case:
    print(f"Case Details:")
    print(f"Display Name: {case.display_name}")
    print(f"Priority: {case.priority}")
    print(f"SOAR Case ID: {case.soar_platform_info.case_id}")
```

The `CaseList` class provides helper methods for working with cases:
- `get_case(case_id)`: Look up a specific case by ID
- `filter_by_priority(priority)`: Get cases with specified priority
- `filter_by_status(status)`: Get cases with specified status
- `filter_by_stage(stage)`: Get cases with specified stage

## Alerts and Case Management

You can retrieve alerts and their associated cases:

```python
from datetime import datetime, timedelta, timezone
from secops import SecOpsClient

client = SecOpsClient()
chronicle = client.chronicle(
    customer_id="your-customer-id",
    project_id="your-project-id"
)

# Get alerts from the last 24 hours
end_time = datetime.now(timezone.utc)
start_time = end_time - timedelta(hours=24)

# Get non-closed alerts
alerts = chronicle.get_alerts(
    start_time=start_time,
    end_time=end_time,
    snapshot_query='feedback_summary.status != "CLOSED"',
    max_alerts=1000
)

# Get alerts from the response
alert_list = alerts.get('alerts', {}).get('alerts', [])

# Extract case IDs from alerts
case_ids = {alert.get('caseName') for alert in alert_list if alert.get('caseName')}

# Get details for cases if any found
if case_ids:
    cases = chronicle.get_cases(list(case_ids))
    
    # Process cases and their related alerts
    for case in cases.cases:
        print(f"Case: {case.display_name}")
        print(f"Priority: {case.priority}")
        print(f"Stage: {case.stage}")
        print(f"Status: {case.status}")
        
        # Find alerts for this case
        case_alerts = [
            alert for alert in alert_list
            if alert.get('caseName') == case.id
        ]
        print(f"Related Alerts: {len(case_alerts)}")
        
        # Find high severity alerts
        high_sev_alerts = [
            alert for alert in case_alerts
            if alert.get('feedbackSummary', {}).get('severityDisplay') == 'HIGH'
        ]
        if high_sev_alerts:
            print(f"High Severity Alerts: {len(high_sev_alerts)}")
```

#### Alert Response Details
The alerts response includes:
- Progress status (0-100%)
- Completion status
- Alert counts (baseline and filtered)
- Alert details including:
  - Rule information
  - Detection details
  - Creation time
  - Case association
  - Feedback summary (status, priority, severity)

#### Case Details
Cases include:
- Display name
- Priority level
- Current stage
- Status
- Associated alerts

You can filter alerts using the snapshot query parameter, which supports fields like:
- detection.rule_set
- detection.rule_name
- detection.alert_state
- feedback_summary.verdict
- feedback_summary.priority
- feedback_summary.severity_display
- feedback_summary.status

## License

This project is licensed under the Apache License 2.0 - see the LICENSE file for details.