PostgreSQL comes with a simple non-durable topic-based publish-subscribe notification system. It’s no Kafka, but the features do support common use cases.

The notification consists of a topic name and a payload (upto about 8000 characters). The payload would typically be a JSON string, but of course it can be anything. You can send a notification using the [NOTIFY](https://www.postgresql.org/docs/current/static/sql-notify.html) command:

```sql
NOTIFY 'foo_events', '{"userid":42,"action":"grok"}'
```