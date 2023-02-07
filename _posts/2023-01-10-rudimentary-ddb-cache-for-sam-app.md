---
layout: post
title:  "DynamoDB: Rudimentary DDB Cache for Serverless App"
date:   2023-01-10 11:57:04 +0000
# categories: sam, dynamodb, cache, testing, moto
---

I'm currently using an AWS SAM application which has started to get 429 throttling responses from various AWS services.

This is a (very) rudimentary cache for DynamoDB which I'm using to reduce the number of calls to Cognito and S3.

## Implementation

Here is a rudiementary cache for DynamoDB:

```python
from datetime import datetime
from datetime import timedelta

import boto3

from config import Config


class CacheService:
    """CacheService is a class that provides a cache service for the application using DynamoDB"""

    def __init__(self):
        """initialize the CacheService class"""
        self.config = Config()
        self.app_env = self.config.app_env
        self.ddb = boto3.client("dynamodb")
        self.cache_table = "%s-cache-%s" % (self.config.app_name, self.config.app_env)

    def set(self, key: str, value: str, ttl_seconds: int = 3300) -> None:
        """set a key value pair in the cache with expire"""
        """Default TTL is 55 minutes"""
        ttl = self.set_ttl(ttl_seconds)
        self.ddb.update_item(
            TableName=self.cache_table,
            Key={"id": {"S": key}},
            UpdateExpression="SET cached_value = :cached_value, expires_at = :expires_at",
            ExpressionAttributeValues={
                ":cached_value": {"S": value},
                ":expires_at": {"N": str(ttl.timestamp())},
            },
        )

    def get(self, key: str) -> str or None:
        """get a key value pair from the cache"""
        response = self.ddb.get_item(TableName=self.cache_table, Key={"id": {"S": key}})
        if "Item" in response:
            item = response["Item"]
            if "cached_value" in item and "expires_at" in item:
                if not self.has_expired(float(item["expires_at"]["N"])):
                    return item["cached_value"]["S"]
        return None

    def set_ttl(self, ttl_seconds: int = 3300) -> datetime:
        """set the ttl for the cache, now plus ttl_seconds"""
        return datetime.now() + timedelta(seconds=ttl_seconds)

    def has_expired(self, expire: float) -> bool:
        """check if the cache has expired"""
        return datetime.now().timestamp() > expire

```

And here are the tests:

```python
from datetime import datetime
from datetime import timedelta
from unittest import TestCase

import boto3
from moto import mock_dynamodb2
from service.cache_service import CacheService

from config import Config


@mock_dynamodb2
class TestCacheService(TestCase):
    def setUp(self):
        self.config = Config()
        self.cache_table = "%s-cache-%s" % (self.config.app_name, self.config.app_env)
        self.table = self.create_ddb_table()

    def tearDown(self):
        self.table.delete()

    def create_ddb_table(self):
        db = boto3.resource("dynamodb")
        db.create_table(
            TableName=self.cache_table,
            KeySchema=[{"AttributeName": "id", "KeyType": "HASH"}],
            AttributeDefinitions=[{"AttributeName": "id", "AttributeType": "S"}],
            BillingMode="PAY_PER_REQUEST",
        )

        return db.Table(self.cache_table)

    def test_set_data(self):
        cacheService = CacheService()
        cacheService.set("test-key", "test-value")
        response = self.table.get_item(Key={"id": "test-key"})
        self.assertEqual(response["Item"]["cached_value"], "test-value")

    def test_set_update_with_expired_ttl(self):
        cacheService = CacheService()
        cacheService.set("test-key", "test-value", -1)
        get_none = cacheService.get("test-key")
        self.assertEqual(get_none, None)
        cacheService.set("test-key", "test-value")
        get_value = cacheService.get("test-key")
        self.assertEqual(get_value, "test-value")

    def test_ttl_data(self):
        cache = CacheService()
        ttl = cache.set_ttl()
        expected = datetime.now() + timedelta(seconds=3300)
        self.assertAlmostEqual(ttl.timestamp(), expected.timestamp(), delta=1)

    def test_has_expired(self):
        cache = CacheService()
        ttl = cache.set_ttl()
        self.assertFalse(cache.has_expired(ttl.timestamp()))

    def test_has_not_expired(self):
        cache = CacheService()
        ttl = cache.set_ttl()
        self.assertTrue(cache.has_expired(ttl.timestamp() - 3600))

    def test_get_data(self):
        cacheService = CacheService()
        cacheService.set("test-key", "test-value")
        response = cacheService.get("test-key")
        self.assertEqual(response, "test-value")

    def test_get_expired(self):
        cacheService = CacheService()
        cacheService.set("test-key", "test-value", -1)
        response = cacheService.get("test-key")
        self.assertEqual(response, None)
```

And here is an example implmentation of the cache service:

```python
import logging
import time

import requests
from requests.auth import HTTPBasicAuth
from service.cache_service import CacheService

from config import Config


class RefreshToken:
    def __init__(self):
        self.config = Config()
        self.cache = CacheService()

    def get_token(self):
        """get the access token from the cache or from the auth server"""
        token = self.cache.get("access_token")
        if token is None:
            token = self.get_token_from_auth_server()
            self.cache.set("access_token", token)
        return token
```

I'd like to include a `TimeToLiveSpecification` against the table itself, but for now, i've kept it super simple:

```yaml
  CacheTable:
    Type: AWS::Serverless::SimpleTable
    Properties:
      TableName: !Sub "${AppName}-cache-${AppEnv}"
      SSESpecification:
        SSEEnabled: true
```

Plenty of room for improvement here, but it's gets us past the 429 throttling errors, we can take a breath and think about how to improve it.

Idea 1: Catch throttling errors (or any error) and respond accordingly. Perhaps a throttling error from DynamoDB should return a None, or back off and retry n times before returning a None.
