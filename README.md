# Notification Service

## Overview

The Notification Service is responsible for delivering notifications to users through multiple channels:

- Email
- SMS
- Push Notifications

The system is designed to support **1M+ users** while guaranteeing:

- No duplicate notifications
- No missed notifications
- High availability
- Provider failover
- Horizontal scalability
- Auditability and traceability

---

# Requirements

## Functional Requirements

- Send notifications through Email, SMS, and Push channels.
- Support transactional and marketing notifications.
- Support user notification preferences.
- Allow multiple delivery providers per channel.
- Support retry and failover mechanisms.
- Track notification delivery lifecycle.

## Non-Functional Requirements

- Support millions of notifications per day.
- Ensure idempotent processing.
- Ensure eventual delivery when providers are temporarily unavailable.
- Support monitoring and alerting.
- Provide full audit trail for all notifications.

---

# High-Level Architecture

                              ┌─────────────────┐
                              │ Django Services │
                              └────────┬────────┘
                                       │
                                       ▼
                              ┌─────────────────┐
                              │ Notification API│
                              └────────┬────────┘
                                       │
                                       ▼
                              ┌─────────────────┐
                              │ PostgreSQL      │
                              │ Notifications   │
                              └────────┬────────┘
                                       │
                                 Outbox Pattern
                                       │
                                       ▼
                              ┌─────────────────┐
                              │ Celery Broker   │
                              │ Redis/RabbitMQ  │
                              └────────┬────────┘
                                       │
                ┌───--─────────────────┼──────────----------──────────┐
                │                      │                              │
                ▼                      ▼                              ▼
       ┌─────────────┐           ┌─────────────┐               ┌─────────────┐
       │Email Worker │           │ SMS Worker  │               │Push Worker  │
       └──────┬──────┘           └──────┬──────┘               └──────┬──────┘
              │                         │                             │
              ▼                         ▼                             ▼
       SES/SendGrid          Twilio/AfricaTalking                  FCM/APNS

---

# Technology Stack

| Component | Technology |
|------------|------------|
| API | Django REST Framework |
| Database | PostgreSQL |
| Async Processing | Celery |
| Message Broker | RabbitMQ / Redis |
| Cache | Redis |
| Monitoring | Prometheus + Grafana |
| Logging | ELK Stack / OpenSearch |
| Email Providers | AWS SES, SendGrid |
| SMS Providers | Twilio, Africa's Talking |
| Push Providers | Firebase FCM, APNS |

---

# Data Model

## Notification

Represents a notification event.

```python

import uuid
from django.db import models
from django.http import Http404


def generate_id():
    return uuid.uuid4().hex


class BaseAbstractModel(models.Model):
    id = models.CharField(
        primary_key=True, editable=False, default=generate_id, max_length=70
    )
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)


    class Meta:
        abstract = True
        ordering = ("-created_at",)

    def save(self, actor=None, *args, **kwargs):
        if not self.pk:
            self.created_by = actor
        super(BaseAbstractModel, self).save(*args, **kwargs)



class Notification(BaseAbstractModel):
    user = models.ForeignKey(User)
    event_type = models.CharField(max_length=100)

    STATUS_CHOICES = (
        ("pending", "Pending"),
        ("processing", "Processing"),
        ("completed", "Completed"),
        ("failed", "Failed"),
    )

    status = models.CharField(max_length=20)
    payload = models.JSONField()



class NotificationPreference(BaseAbstractModel):
    user = models.OneToOneField(User)

    email_enabled = models.BooleanField(default=True)
    sms_enabled = models.BooleanField(default=True)
    push_enabled = models.BooleanField(default=True)

    marketing_enabled = models.BooleanField(default=False)

```


## Idempotency for notification request.

This ensures no duplicate notification is sent in cases of network retries, worker crashes etc.

```
class NotificationRequest(models.Model):
    idempotency_key = models.CharField(
        max_length=255,
        unique=True
    )

    notification = models.ForeignKey(
        Notification,
        on_delete=models.CASCADE
    )

```
---


# Outbox Pattern

Problem

Without an outbox:

1. Notification saved in database.
2. Queue publish fails.
3. Notification never gets processed.

Solution

Use Outbox Table.

```
class NotificationOutbox(BaseAbstractModel):
    notification = models.ForeignKey(Notification)
    published = models.BooleanField(default=False)
```

Transaction, this ensures the process is atomic

```
with transaction.atomic():
    notification = Notification.objects.create(...)
    NotificationOutbox.objects.create(
        notification=notification
    )
```



---

# Processing Flow

* Step 1

       Application triggers notification.

       Example:
              NotificationService.send(
                     user=user,
                     event_type="withdrawal_successful",
                     channels=["email", "sms"]
              )

* Step 2

       Notification record created.

* Step 3

       Outbox entry created.

* Step 4

       Publisher pushes message to broker.

* Step 5

       Worker receives task.

* Step 6

       Worker creates delivery record.

* Step 7

       Provider receives notification.

* Step 8

       Delivery status updated.



##
# Provider Abstraction
Also, for cases where we are using multiple providers, I would use Abstraction with Factory to switch between providers in cases of a provider failure while recording the provider used, the provider that failed, the count of trials etc.

For example

```
# provider_base.py
from abc import ABC, abstractmethod


class BaseSMSProvider(ABC):
    """Abstract interface for all SMS providers."""

    @abstractmethod
    def send_sms(self, phone_number, message: str) -> dict:
        """Send a message and return the response."""
        pass



#provider_integration

from .provider_base import BaseSMSProvider
from twilio.rest import Client


class TwilioSMSProvider(BaseSMSProvider):
    def send_sms(self, phone_number: str, message: str):
       ...


#Factory
... imports ...

class SMSProviderFactory:
    _providers = {
        "twilio": TwilioSMSProvider,
        "termii": TermiSMSProivder,
        "africas_talking": AfricaIsTalkingProvider,
        "kudi_sms": KudiSMSProvider,
    }

    @classmethod
    def get_provider(cls, provider_name: str):
        provider_cls = cls._providers.get(provider_name.lower())
        if not provider_cls:
            raise ValueError(f"Unsupported SMS provider: {provider_name}")
        return provider_cls()



Psudo-Code

if provider.is_healthy():
    provider.send(...)
else:
    fallback_provider.send(...)

```

So we use the factory to determine which provider is selected and in cases of failure, switch between providers.


---

In addition,  I'd include in  a **webhook reconciliation section**. For example, SES, SendGrid, Twilio, and FCM may accept a notification and later report delivery/bounce/failure asynchronously. I'd add webhook endpoints that update `NotificationDelivery.status` so the system's final state reflects the provider's actual delivery outcome rather than just the send attempt. This becomes very important at 1M+ user scale.

Thank you. 