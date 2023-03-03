---
layout: post
title: "Cloudwatch Dashboards for SAM: Monitoring Functions"
date: 2023-02-22 06:44:08 +0000
# categories: aws, services, insight
---

I was guilty of creating resources using AWS SAM and then, once in production, forgetting about them and moving onto the next feature.

This was a problem - I had no idea how well my resources are performing, if they could be performing better, or if they are even being used at all.

The following is an example of how to create a dashboard for functions within a SAM application using a CustomResource and a configuration YAML file.

There are costs to a dashboard - but the $5 a month you spend here is better than over spending on resources which are not being used, or are over provisioned.

## Dashbaord Sections

In my world, each new feature is based on an ADR, or a (bug/story) ticket, or a github issue. What I wanted is a way of creating a dashboard and dividing the dashboard by each piece of work.

## Implementation

This is the sort of YAML file I wanted to use to help create the dashboard:

```yaml
dashboards:
  - name: app-overview
    title: User Birthday Microserive 
    description: A Microservice which gets a users birthday
    github: https://github.com/speakcloud/ms-users-birthday
    sections:
      - title: Store
        description: Stores a users birthday into a DynamoDB table
        document_type: ADR
        document_link: https://github.com/speakclouder/ms-users-birtday/blob/mainline/docs/adr/002-store.md
        functions:
          - name: ms-birthday-store
          - name: ms-birthday-delete
          - name: ms-birthday-check
```

I would update the YAML file with each new feature by adding a new "section". At the moment, I just wanted to do functins, but it could be extended to include API Gateway endpoints, S3 buckets, SQS etc.

The following is the code for the CustomResource:

`monitoring_handler.py`

```python
from monitoring_service import MonitoringService
from pathlib import Path
import os
import yaml
import logging
import traceback
import boto3
import cfnresponse


def handle(event, context):
    try:
        location = event['ResourceProperties']['Location']
        app_name = event['ResourceProperties']['AppName']
        env = event['ResourceProperties']['AppEnv']

        config = get_dashboard_config(location)

        monitoring = MonitoringService(config, env, app_name)

        if event['RequestType'] == 'Delete':
            response = monitoring.delete_cloudwatch_dashboards()
            return cfnresponse.send(event, context, cfnresponse.SUCCESS, response)

        responses = monitoring.create_cloudwatch_dashboards()

        return cfnresponse.send(event, context, cfnresponse.SUCCESS, responses)
    except Exception as exception:
        traceback.print_exc()
        response_data = {}
        response_data['status'] = 'failure'
        response_data['errorMessage'] = str(exception)
        return cfnresponse.send(event, context, cfnresponse.FAILED, response_data)


def get_dashboard_config(location: str or None) -> dict:
    dashboard_config_file = Path('./monitoring/dashboard.yaml')
    if location:
        dashboard_config_file = Path(location)
    dashboard_config = load_dashboard_config_yaml(dashboard_config_file)
    return dashboard_config


def load_dashboard_config_yaml(dashboard_config_file: str) -> dict:
    with open(dashboard_config_file, 'r') as stream:
        try:
            dashboard_config = yaml.safe_load(stream)
        except yaml.YAMLError as exc:
            logging.exception(exc)
    return dashboard_config
```

`monitoring_service.py`

```python
from pathlib import Path
import logging
import json
import copy
import boto3


class MonitoringService:
    def __init__(self, config: dict, env: str, app_name: str):
        self.config = config
        self.env = env
        self.app_name = app_name
        self.header, self.section, self.function = self.get_raw_config_files()

    def create_cloudwatch_dashboards(self):
        client = boto3.client('cloudwatch')
        dashboard_response = []
        for dashboard, widgets in self.generate_dashboard_json().items():
            dashboard_name = f'{self.app_name}-{dashboard}-{self.env}'
            response = client.put_dashboard(DashboardName=dashboard_name, DashboardBody=json.dumps(widgets[0]))
            dashboard_response.append(response)
        return dashboard_response

    def generate_dashboard_json(self) -> dict:
        dashboards = {}
        for dashboard in self.config['dashboards']:
            dashboards[dashboard['name']] = []

            dashboard_json = self.dashboard_header(dashboard)

            for section in dashboard['sections']:
                dashboard_json['widgets'].append(self.dashboard_sections(section))

                for function in section['functions']:
                    function_metrics = self.dashboard_functions(function)
                
                # Here is where I would add API Gateway metrics, S3 metrics, SQS metrics etc.

                    for metric in function_metrics:
                        dashboard_json['widgets'].append(metric)

            dashboards[dashboard['name']].append(dashboard_json)
        return dashboards

    def get_raw_config_files(self):
        with open(Path('./monitoring/metrics_dashboard.json'), 'r') as dashboard_header:
            raw_header = json.loads(dashboard_header.read())

        with open(Path('./monitoring/metrics_section.json'), 'r') as dashboard_section:
            raw_section = json.loads(dashboard_section.read())

        with open(Path('./monitoring/metrics_function.json'), 'r') as dashboard_function:
            raw_function = json.loads(dashboard_function.read())

        return raw_header, raw_section, raw_function

    def dashboard_header(self, dashboard: dict):
        raw_dashboard = copy.deepcopy(self.header)
        title = raw_dashboard['widgets'][0]['properties']['markdown']
        title = title.replace('TITLE', dashboard['title'])
        title = title.replace('NAME', dashboard['name'])
        title = title.replace('ENVIRONMENT', self.env)
        title = title.replace('DESCRIPTION', dashboard['description'])
        title = title.replace('GITHUB', dashboard['github'])
        raw_dashboard['widgets'][0]['properties']['markdown'] = title
        return raw_dashboard

    def dashboard_sections(self, section: dict):
        raw_section = copy.deepcopy(self.section)
        section_markdown = raw_section['properties']['markdown']
        section_markdown = section_markdown.replace('TITLE', section['title'])
        section_markdown = section_markdown.replace('DESCRIPTION', section['description'])
        section_markdown = section_markdown.replace('ADR', section['ADR'])
        raw_section['properties']['markdown'] = section_markdown
        return raw_section

    def dashboard_functions(self, function: dict) -> list:
        function_copy = json.dumps(copy.deepcopy(self.function))
        function_copy = function_copy.replace('NAME', function['name'])
        function_copy = function_copy.replace('ENVIRONMENT', self.env)
        function_copy = json.loads(function_copy)
        return function_copy

    def delete_cloudwatch_dashboards(self):
        client = boto3.client('cloudwatch')
        dashboards = []
        for dashboard in self.generate_dashboard_json():
            dashboard_name = f'{self.app_name}-{dashboard}-{self.env}'
            dashboards.append(dashboard_name)
        return client.delete_dashboards(DashboardNames=dashboards)

```

And my metric json files would look like this:

`metric_dashboard.json`

```json
{
  "widgets": [
    {
      "height": 2,
      "properties": {
        "background": "solid",
        "markdown": "# TITLE (ENVIRONMENT)\n\nLink: [NAME-ENVIRONMENT](#dashboards:name=NAME-ENVIRONMENT) Github: [GITHUB](GITHUB)\n\nDESCRIPTION"
      },
      "type": "text",
      "width": 24
    }
  ]
}

```

`metric_section.json`

```json
{
  "height": 3,
  "properties": {
    "markdown": "## TITLE\n\nDESCRIPTION\n\n[button:primary:DOCUMENTATION_NAME](DOCUMENTATION_LINK) "
  },
  "type": "text",
  "width": 24
}
```

`metric_function.json`

```json
[
  {
    "height": 6,
    "properties": {
      "metrics": [
        [
          "AWS/Lambda",
          "Duration",
          "FunctionName",
          "NAME-ENVIRONMENT"
        ]
      ],
      "period": 300,
      "region": "eu-west-1",
      "stacked": false,
      "stat": "Average",
      "title": "NAME duration",
      "view": "timeSeries"
    },
    "type": "metric",
    "width": 8
  },
  {
    "height": 6,
    "properties": {
      "metrics": [
        [
          "AWS/Lambda",
          "Invocations",
          "FunctionName",
          "NAME-ENVIRONMENT",
          {
            "color": "#69ae34"
          }
        ]
      ],
      "period": 300,
      "region": "eu-west-1",
      "stacked": false,
      "stat": "Sum",
      "title": "NAME invocations",
      "view": "timeSeries"
    },
    "type": "metric",
    "width": 8
  },
  {
    "height": 6,
    "properties": {
      "metrics": [
        [
          "AWS/Lambda",
          "Invocations",
          "FunctionName",
          "NAME-ENVIRONMENT",
          {
            "color": "#fe6e73",
            "visible": false
          }
        ],
        [
          ".",
          "Errors",
          ".",
          ".",
          "Resource",
          "NAME-ENVIRONMENT"
        ]
      ],
      "period": 300,
      "region": "eu-west-1",
      "stacked": false,
      "stat": "Sum",
      "title": "NAME errors",
      "view": "timeSeries"
    },
    "type": "metric",
    "width": 8
  },
  {
    "height": 3,
    "properties": {
      "query": "SOURCE '/aws/lambda/NAME-ENVIRONMENT' | filter @type = \"REPORT\"\n| stats max(@memorySize / 1024 / 1024) as provisonedMemMB,\nmin(@maxMemoryUsed / 1024 / 1024) as smallestMemReqMB,\navg(@maxMemoryUsed / 1024 / 1024) as avgMemUsedMB,\nmax(@maxMemoryUsed / 1024 / 1024) as maxMemUsedMB,\nprovisonedMemMB - maxMemUsedMB as overProvisionedMB",
      "region": "eu-west-1",
      "stacked": false,
      "title": "NAME over provisioned memory",
      "view": "table"
    },
    "type": "log"
  },
  {
    "height": 3,
    "properties": {
      "query": "SOURCE '/aws/lambda/NAME-ENVIRONMENT' | fields Timestamp, Message\n| filter Message like /ERROR/\n| sort @timestamp desc\n| limit 5",
      "region": "eu-west-1",
      "stacked": false,
      "title": "NAME last 5 errors",
      "view": "table"
    },
    "type": "log"
  },
  {
    "height": 3,
    "properties": {
      "query": "SOURCE '/aws/lambda/NAME-ENVIRONMENT' | filter @type = \"REPORT\"\n| fields @requestId, (@billedDuration/1000) as seconds\n| sort by @billedDuration desc\n| limit 5",
      "region": "eu-west-1",
      "stacked": false,
      "title": "NAME top 5 longest running",
      "view": "table"
    },
    "type": "log"
  },
  {
    "height": 3,
    "properties": {
      "query": "SOURCE '/aws/lambda/NAME-ENVIRONMENT' | filter @message like /Task timed out/\n| stats count() by bin(30m)",
      "region": "eu-west-1",
      "stacked": false,
      "title": "NAME timeouts",
      "view": "table"
    },
    "type": "log"
  }
]
```

I'd then include the fuction in my SAM template like this:


```YAML
  MonitoringFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: handler_monitoring.handle
      CodeUri: ./functions/MonitoringFunction
      Description: Monitoring Stack Resources
      FunctionName: !Sub "${AppName}-monitoring-${AppEnv}"
      Role: !GetAtt LambdaExecutionRole.Arn

  MonitoringFunctionRunner:
    Type: AWS::CloudFormation::CustomResource
    Version: "1.0"
    Properties:
      ServiceToken: !GetAtt MonitoringFunction.Arn
      Region: !Ref AWS::Region
      Location: "./monitoring/dashboard.yaml"
      AppName: !Ref AppName
      AppEnv: !Ref AppEnv
```

The handler would run on stack update, updating to the latest config file.

I need a ton more widgets for different resources - API Gateway endpoints, stores, queues, etc. - but this is a good start. Checkout the `monitoring` package for CDK that does this all for you.

I am likely going to make this all into a package you can pull down. 

By the way, heres some tests:

`test_handler.py`

```python
from mock import patch
from pathlib import Path
from handler_monitoring import handle
from handler_monitoring import get_dashboard_config
from handler_monitoring import load_dashboard_config_yaml


def test_loads_dashboard_yaml():
    dashboard_config_file = Path('./tests/_fixtures/dashboard.yaml')
    dashboard_config = get_dashboard_config(dashboard_config_file)
    assert dashboard_config['dashboards'][0]['title'] == 'MS Event Conductor'
    assert dashboard_config['dashboards'][0]['description'] == 'MS Event Conductor Resource Dashboard'
    assert len(dashboard_config['dashboards'][0]['sections']) == 1
    assert dashboard_config['dashboards'][0]['sections'][0]['title'] == 'Store'
    assert dashboard_config['dashboards'][0]['sections'][0]['description'] == 'Dumps all events into Stratus Aggregate'
    assert len(dashboard_config['dashboards'][0]['sections'][0]['functions']) == 1
    assert dashboard_config['dashboards'][0]['sections'][0]['functions'][0]['name'] == 'birthday-micro-service-store'
```


`test_service.py`

```python
from mock import patch
import json
import yaml
from pathlib import Path
from monitoring_service import MonitoringService
from moto import mock_cloudwatch

env = 'local'
app_name = 'birthday-micro-service'

with open('./tests/_fixtures/dashboard.yaml', 'r') as stream:
    config = yaml.safe_load(stream)


with open('./tests/_fixtures/dashboard_bigger.yaml', 'r') as stream:
    bigger_config = yaml.safe_load(stream)


def test_service_loads_config():
    monitoring_service = MonitoringService(config, env, app_name)
    assert monitoring_service.config['dashboards'][0]['title'] == 'MS Event Conductor'
    assert monitoring_service.config['dashboards'][0]['description'] == 'MS Event Conductor Resource Dashboard'
    assert len(monitoring_service.config['dashboards'][0]['sections']) == 1
    assert monitoring_service.config['dashboards'][0]['sections'][0]['title'] == 'Store'
    assert monitoring_service.config['dashboards'][0]['sections'][0]['description'] == 'Dumps all events into Stratus Aggregate'
    assert len(monitoring_service.config['dashboards'][0]['sections'][0]['functions']) == 1
    assert monitoring_service.config['dashboards'][0]['sections'][0]['functions'][0]['name'] == 'birthday-micro-service-store'


def test_service_loads_raw_config_files():
    monitoring_service = MonitoringService(config, env, app_name)
    assert 'TITLE' in monitoring_service.header['widgets'][0]['properties']['markdown']
    assert 'DOCUMENTATION_NAME' in monitoring_service.section['properties']['markdown']
    assert 'AWS/Lambda' in monitoring_service.function[0]['properties']['metrics'][0]


def test_generates_dashboard():
    monitoring_service = MonitoringService(config, env, app_name)
    dash = monitoring_service.generate_dashboard_json()
    assert len(dash['birthday-micro-service'][0]['widgets']) == 9


def test_handlers_multple_sections():
    monitoring_service = MonitoringService(bigger_config, env, app_name)
    dashboard = monitoring_service.generate_dashboard_json()
    assert len(dashboard['birthday-micro-service'][0]['widgets']) == 24


@mock_cloudwatch
def test_creates_dashboard():
    responses = MonitoringService(bigger_config, env, app_name).create_cloudwatch_dashboards()
    assert responses[0]['ResponseMetadata']['HTTPStatusCode'] == 200


@mock_cloudwatch
def test_deletes_dashboard():
    responses = MonitoringService(bigger_config, env, app_name).create_cloudwatch_dashboards()
    assert responses[0]['ResponseMetadata']['HTTPStatusCode'] == 200
    response = MonitoringService(bigger_config, env, app_name).delete_cloudwatch_dashboards()
    assert response['ResponseMetadata']['HTTPStatusCode'] == 200
```