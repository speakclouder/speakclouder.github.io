---
layout: post
title:  "Testing Event Bridge put_events with Moto"
date:   2022-01-02 14:43:24 +0000
# categories: testing, eventbridge, moto
---

This post describes how I tested a Python Lambda Function doing a put_event to EventBridge using moto

## Steps

I have a python lambda where the Lambda does a little process of some data, then puts a "complete" event onto a custom EventBridge bus.

The function looked something like this:

```python
import traceback
import boto3
import json


def handle(event, context=None):    
    response = do_something(event)
    send_event_when_something_done(response)


def send_event_when_something_done(response: dict) -> None:
    """Send event to custom bus"""
    client = boto3.client('events')
    client.put_events(
        Entries=[
            {
                'Source': 'do-something-function',
                'DetailType': 'do-something-function.complete',
                'Detail': json.dumps(response),
                'EventBusName': 'my-custom-bus',
            },
        ],
    )
```

I wanted to make sure that the event was getting put onto the bus as expected. To do this, I needed to create a fake world in moto:

1. Create an SQS queue
2. Create a custom event bus
3. Create a rule which puts the event I am testing onto the SQS queue
4. fire the handler, and make sure the event looked like what  


```python
import pytest
import boto3
from moto import mock_events
from moto import mock_sqs

from handler_do_something import handle


@mock_events
@mock_sqs
def test_gets_traceback():
    sqs = boto3.resource('sqs', region_name='eu-west-1')
    queue = sqs.create_queue(QueueName='test-queue-local')

    client = boto3.client('events', region_name='eu-west-1')
    client.create_event_bus(Name='test-bus')

    client.put_rule(
        Name='do-something-function-onto-queue',
        EventPattern='{"source": ["do-something-function"]}',
        State='ENABLED',
        EventBusName='test-bus',
    )

    client.put_targets(
        Rule='do-something-function',
        EventBusName='test-bus',
        Targets=[
            {
                'Id': '1',
                'Arn': queue.attributes['QueueArn'],
            },
        ],
    )

    handle({"do_this": "123"})

    message = json.loads(queue.receive_messages()[0].body)

    expected = {
        'version': '0',
        'detail-type': 'do-something-function.complete',
        'source': 'do-something-function',
        'account': '123456789012',
        'region': 'eu-west-1',
        'resources': [],
        'detail': {'do_this': '123', 'status': 'complete'}
    }

    for key, value in expected.items():
        if key in ['id', 'time']:
            continue
        assert message[key] == expected[key]
```

I now know that the message that finds it's way to the SQS queue is exactly how I want it to look.