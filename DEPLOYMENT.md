# Clinic Platform Deployment Guide

## RabbitMQ Message Broker Deployment

This guide provides comprehensive instructions for deploying RabbitMQ as the message broker for the Clinic platform. RabbitMQ enables reliable, asynchronous communication between microservices in the healthcare system.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation Methods](#installation-methods)
- [Configuration](#configuration)
- [Integration with Clinic Platform](#integration-with-clinic-platform)
- [Common Use Cases](#common-use-cases)
- [Security Configuration](#security-configuration)
- [Monitoring and Management](#monitoring-and-management)
- [Troubleshooting](#troubleshooting)
- [Production Considerations](#production-considerations)

## Prerequisites

### System Requirements

- **Operating System**: Linux (Ubuntu 20.04+ recommended), Windows Server 2019+, or macOS 10.15+
- **Memory**: Minimum 2GB RAM (4GB+ recommended for production)
- **Disk Space**: Minimum 10GB free space
- **Network**: Ports 5672 (AMQP), 15672 (Management UI), 25672 (Inter-node communication)

### Version Requirements

- **RabbitMQ**: Version 3.11.0 or higher (latest stable recommended)
- **Erlang/OTP**: Version 25.0 or higher (automatically managed in most installations)
- **Docker**: Version 20.10+ (if using containerized deployment)
- **Docker Compose**: Version 2.0+ (for multi-container setups)

## Installation Methods

### Method 1: Docker Deployment (Recommended)

This is the recommended approach for development and production environments.

#### Single Container Setup

```bash
# Pull the official RabbitMQ image with management plugin
docker pull rabbitmq:3.11-management

# Run RabbitMQ container
docker run -d \
  --name clinic-rabbitmq \
  --hostname clinic-rabbit \
  -p 5672:5672 \
  -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=clinic_admin \
  -e RABBITMQ_DEFAULT_PASS=secure_password_123 \
  -v rabbitmq_data:/var/lib/rabbitmq \
  rabbitmq:3.11-management
```

#### Docker Compose Setup

Create a `docker-compose.rabbitmq.yml` file:

```yaml
version: '3.8'

services:
  rabbitmq:
    image: rabbitmq:3.11-management
    container_name: clinic-rabbitmq
    hostname: clinic-rabbit
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: clinic_admin
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASSWORD}
      RABBITMQ_DEFAULT_VHOST: clinic
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
      - ./config/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf
    restart: unless-stopped
    networks:
      - clinic-network

volumes:
  rabbitmq_data:

networks:
  clinic-network:
    driver: bridge
```

Deploy with:
```bash
# Set environment variable
export RABBITMQ_PASSWORD=your_secure_password

# Start RabbitMQ
docker-compose -f docker-compose.rabbitmq.yml up -d
```

### Method 2: Native Installation

#### Ubuntu/Debian

```bash
# Update package index
sudo apt-get update

# Install dependencies
sudo apt-get install curl gnupg apt-transport-https -y

# Add RabbitMQ signing key
curl -1sLf "https://keys.openpgp.org/vks/v1/by-fingerprint/0A9AF2115F4687BD29803A206B73A36E6026DFCA" | sudo gpg --dearmor | sudo tee /usr/share/keyrings/com.rabbitmq.team.gpg > /dev/null

# Add RabbitMQ repository
echo "deb [signed-by=/usr/share/keyrings/com.rabbitmq.team.gpg] https://packagecloud.io/rabbitmq/rabbitmq-server/ubuntu/ $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/rabbitmq.list

# Update package index
sudo apt-get update

# Install RabbitMQ
sudo apt-get install rabbitmq-server -y

# Enable and start service
sudo systemctl enable rabbitmq-server
sudo systemctl start rabbitmq-server

# Enable management plugin
sudo rabbitmq-plugins enable rabbitmq_management
```

#### CentOS/RHEL

```bash
# Install EPEL repository
sudo yum install epel-release -y

# Add RabbitMQ repository
curl -s https://packagecloud.io/install/repositories/rabbitmq/rabbitmq-server/script.rpm.sh | sudo bash

# Install RabbitMQ
sudo yum install rabbitmq-server -y

# Enable and start service
sudo systemctl enable rabbitmq-server
sudo systemctl start rabbitmq-server

# Enable management plugin
sudo rabbitmq-plugins enable rabbitmq_management
```

## Configuration

### Basic Configuration

Create `/etc/rabbitmq/rabbitmq.conf` for native installations or mount as volume for Docker:

```ini
# Listener configuration
listeners.tcp.default = 5672
management.tcp.port = 15672

# Logging
log.console = true
log.console.level = info
log.file.level = info

# Memory and disk space thresholds
vm_memory_high_watermark.relative = 0.6
disk_free_limit.relative = 2.0

# Heartbeat settings
heartbeat = 60

# Default virtual host
default_vhost = clinic
default_user = clinic_admin
default_pass = secure_password_123

# Security settings
loopback_users.guest = false
```

### Environment-Specific Configuration

#### Development Environment
```ini
# More verbose logging for development
log.console.level = debug
log.file.level = debug

# Lower memory thresholds for development
vm_memory_high_watermark.relative = 0.8
```

#### Production Environment
```ini
# Optimized for production
vm_memory_high_watermark.relative = 0.4
disk_free_limit.relative = 5.0

# Enable clustering (if using multiple nodes)
cluster_formation.peer_discovery_backend = rabbit_peer_discovery_classic_config
cluster_formation.classic_config.nodes.1 = rabbit@clinic-rabbit-1
cluster_formation.classic_config.nodes.2 = rabbit@clinic-rabbit-2
cluster_formation.classic_config.nodes.3 = rabbit@clinic-rabbit-3
```

## Integration with Clinic Platform

### Connection Configuration

The Clinic platform connects to RabbitMQ using these standard parameters:

```json
{
  "host": "localhost",
  "port": 5672,
  "virtual_host": "clinic",
  "username": "clinic_admin",
  "password": "secure_password_123",
  "connection_timeout": 30,
  "heartbeat": 60
}
```

### Virtual Hosts and Users

Set up dedicated resources for the Clinic platform:

```bash
# Create virtual host for clinic
sudo rabbitmqctl add_vhost clinic

# Create clinic application user
sudo rabbitmqctl add_user clinic_app app_password_123

# Set permissions for clinic app user
sudo rabbitmqctl set_permissions -p clinic clinic_app ".*" ".*" ".*"

# Create read-only monitoring user
sudo rabbitmqctl add_user clinic_monitor monitor_password_123
sudo rabbitmqctl set_permissions -p clinic clinic_monitor "^$" "^$" ".*"
```

### Exchange and Queue Setup

The Clinic platform uses the following messaging patterns:

```bash
# Declare exchanges for different domain events
rabbitmqadmin declare exchange name=clinic.patients type=topic durable=true
rabbitmqadmin declare exchange name=clinic.appointments type=topic durable=true
rabbitmqadmin declare exchange name=clinic.prescriptions type=topic durable=true
rabbitmqadmin declare exchange name=clinic.notifications type=fanout durable=true

# Declare queues for microservices
rabbitmqadmin declare queue name=patient.service.events durable=true
rabbitmqadmin declare queue name=appointment.service.events durable=true
rabbitmqadmin declare queue name=prescription.service.events durable=true
rabbitmqadmin declare queue name=notification.service.events durable=true

# Bind queues to exchanges with routing keys
rabbitmqadmin declare binding source=clinic.patients destination=patient.service.events routing_key="patient.*"
rabbitmqadmin declare binding source=clinic.appointments destination=appointment.service.events routing_key="appointment.*"
rabbitmqadmin declare binding source=clinic.prescriptions destination=prescription.service.events routing_key="prescription.*"
rabbitmqadmin declare binding source=clinic.notifications destination=notification.service.events
```

## Common Use Cases

### 1. Patient Registration Events

When a new patient registers in the system:

**Publisher (Patient Service)**:
```python
# Example Python code using pika
import pika
import json

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost', 5672, 'clinic'))
channel = connection.channel()

# Publish patient registration event
patient_data = {
    "patient_id": "12345",
    "event_type": "patient.registered",
    "timestamp": "2024-01-15T10:30:00Z",
    "data": {
        "name": "John Doe",
        "email": "john.doe@email.com",
        "phone": "+1234567890"
    }
}

channel.basic_publish(
    exchange='clinic.patients',
    routing_key='patient.registered',
    body=json.dumps(patient_data),
    properties=pika.BasicProperties(delivery_mode=2)  # Make message persistent
)
```

**Consumer (Notification Service)**:
```python
def process_patient_registration(ch, method, properties, body):
    patient_data = json.loads(body)
    # Send welcome email
    send_welcome_email(patient_data['data']['email'])
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_consume(queue='notification.service.events', on_message_callback=process_patient_registration)
```

### 2. Appointment Scheduling

**High-Priority Appointment Notifications**:
```python
# Use priority queues for urgent appointments
channel.queue_declare(queue='urgent.appointments', durable=True, arguments={'x-max-priority': 10})

# Publish urgent appointment with high priority
channel.basic_publish(
    exchange='clinic.appointments',
    routing_key='appointment.urgent',
    body=json.dumps(appointment_data),
    properties=pika.BasicProperties(priority=9, delivery_mode=2)
)
```

### 3. Prescription Processing

**Dead Letter Queue for Failed Prescriptions**:
```python
# Declare main queue with DLX
channel.queue_declare(
    queue='prescription.processing',
    durable=True,
    arguments={
        'x-dead-letter-exchange': 'clinic.dlx',
        'x-dead-letter-routing-key': 'prescription.failed',
        'x-message-ttl': 300000  # 5 minutes TTL
    }
)

# Declare dead letter queue
channel.queue_declare(queue='prescription.failed', durable=True)
```

## Security Configuration

### SSL/TLS Configuration

For production environments, enable SSL/TLS:

```ini
# SSL listener configuration
listeners.ssl.default = 5671
ssl_options.cacertfile = /etc/rabbitmq/ssl/ca-certificate.pem
ssl_options.certfile = /etc/rabbitmq/ssl/server-certificate.pem
ssl_options.keyfile = /etc/rabbitmq/ssl/server-key.pem
ssl_options.verify = verify_peer
ssl_options.fail_if_no_peer_cert = true
```

### Authentication and Authorization

Enable LDAP authentication for enterprise environments:

```ini
# LDAP configuration
auth_backends.1 = rabbit_auth_backend_ldap
auth_backends.2 = rabbit_auth_backend_internal

auth_ldap.servers.1 = ldap.clinic.internal
auth_ldap.user_dn_pattern = uid=${username},ou=users,dc=clinic,dc=internal
auth_ldap.use_ssl = true
```

## Monitoring and Management

### Management UI

Access the RabbitMQ Management UI at `http://localhost:15672` using the admin credentials.

Key metrics to monitor:
- Queue lengths
- Message rates (publish/deliver/ack)
- Connection counts
- Memory usage
- Disk space

### Prometheus Integration

Enable Prometheus metrics plugin:

```bash
sudo rabbitmq-plugins enable rabbitmq_prometheus
```

Add to `rabbitmq.conf`:
```ini
prometheus.tcp.port = 15692
prometheus.path = /metrics
```

### Health Checks

Implement health checks for the Clinic platform:

```bash
# Basic health check
rabbitmqctl ping

# Detailed cluster status
rabbitmqctl cluster_status

# Check if all required queues exist
rabbitmqctl list_queues name | grep -E "(patient|appointment|prescription|notification)"
```

### Automated Monitoring Script

```bash
#!/bin/bash
# clinic-rabbitmq-health.sh

RABBITMQ_HOST="localhost"
RABBITMQ_PORT="15672"
RABBITMQ_USER="clinic_admin"
RABBITMQ_PASS="secure_password_123"

# Check if RabbitMQ is responding
if ! curl -s -u "$RABBITMQ_USER:$RABBITMQ_PASS" \
    "http://$RABBITMQ_HOST:$RABBITMQ_PORT/api/overview" > /dev/null; then
    echo "ERROR: RabbitMQ is not responding"
    exit 1
fi

# Check queue lengths
QUEUE_COUNT=$(curl -s -u "$RABBITMQ_USER:$RABBITMQ_PASS" \
    "http://$RABBITMQ_HOST:$RABBITMQ_PORT/api/queues" | \
    jq '[.[] | select(.messages > 1000)] | length')

if [ "$QUEUE_COUNT" -gt 0 ]; then
    echo "WARNING: $QUEUE_COUNT queues have more than 1000 messages"
fi

echo "RabbitMQ health check passed"
```

## Troubleshooting

### Common Issues and Solutions

#### 1. RabbitMQ fails to start

**Symptoms**: Service won't start, connection refused errors

**Solutions**:
```bash
# Check logs
sudo journalctl -u rabbitmq-server
# or for Docker
docker logs clinic-rabbitmq

# Check if ports are already in use
sudo netstat -tlnp | grep 5672

# Verify Erlang installation
erl -eval 'erlang:display(erlang:system_info(version)), halt().'

# Reset RabbitMQ node (CAUTION: This will delete all data)
sudo rabbitmqctl stop_app
sudo rabbitmqctl reset
sudo rabbitmqctl start_app
```

#### 2. High memory usage

**Symptoms**: VM memory watermark alarms, slow performance

**Solutions**:
```bash
# Check current memory usage
sudo rabbitmqctl status | grep memory

# Set memory alarm manually (temporary)
sudo rabbitmqctl set_vm_memory_high_watermark 0.7

# Purge messages from queues if safe to do so
sudo rabbitmqctl purge_queue patient.service.events
```

#### 3. Connection timeout issues

**Symptoms**: Clients can't connect, timeout errors

**Solutions**:
```bash
# Check firewall rules
sudo ufw status | grep 5672

# Verify RabbitMQ is listening on correct interface
sudo netstat -tlnp | grep 5672

# Test connection from client machine
telnet rabbitmq-server 5672
```

#### 4. Message loss or duplication

**Symptoms**: Missing messages, duplicate processing

**Solutions**:
- Ensure messages are marked as persistent
- Use publisher confirms for critical messages
- Implement idempotent message processing
- Check consumer acknowledgment patterns

```python
# Example: Publisher confirms
channel.confirm_delivery()
if channel.basic_publish(exchange='clinic.patients', routing_key='patient.created', body='...'):
    print("Message delivered successfully")
else:
    print("Message delivery failed")
```

### Performance Tuning

#### For High-Throughput Scenarios

```ini
# Increase file descriptors
# In /etc/systemd/system/rabbitmq-server.service.d/limits.conf
[Service]
LimitNOFILE=65536

# Optimize TCP settings
tcp_listen_options.nodelay = true
tcp_listen_options.sndbuf = 32768
tcp_listen_options.recbuf = 32768

# Increase channel limit
channel_max = 2048
```

#### Queue Configuration

```bash
# Create queues with optimal settings
rabbitmqadmin declare queue name=high.throughput.queue \
  durable=true \
  arguments='{"x-queue-mode":"lazy"}'
```

## Production Considerations

### High Availability Setup

For production environments, implement RabbitMQ clustering:

```bash
# On each node, set the same Erlang cookie
echo "clinic_cluster_cookie" | sudo tee /var/lib/rabbitmq/.erlang.cookie
sudo chmod 400 /var/lib/rabbitmq/.erlang.cookie

# Join nodes to cluster (run on nodes 2 and 3)
sudo rabbitmqctl stop_app
sudo rabbitmqctl join_cluster rabbit@clinic-rabbit-1
sudo rabbitmqctl start_app
```

### Backup and Recovery

```bash
# Export definitions (users, vhosts, exchanges, queues)
rabbitmqadmin export backup.json

# Import definitions
rabbitmqadmin import backup.json

# Backup message data (requires stopping RabbitMQ)
sudo systemctl stop rabbitmq-server
tar -czf rabbitmq-data-backup.tar.gz /var/lib/rabbitmq/
sudo systemctl start rabbitmq-server
```

### Load Balancing

Use HAProxy or similar for load balancing across RabbitMQ nodes:

```
backend rabbitmq_backend
    balance roundrobin
    option tcp-check
    server rabbit1 10.0.1.10:5672 check
    server rabbit2 10.0.1.11:5672 check
    server rabbit3 10.0.1.12:5672 check
```

### Capacity Planning

**Message Rate Guidelines**:
- Small clinic (< 1000 patients): 100-500 messages/second
- Medium clinic (1000-10000 patients): 500-2000 messages/second  
- Large clinic (> 10000 patients): 2000+ messages/second

**Resource Recommendations**:
- **CPU**: 2-4 cores per 1000 messages/second
- **Memory**: 4GB base + 1GB per 10000 queued messages
- **Storage**: SSD recommended, 100GB+ for message persistence

### Deployment Checklist

Before deploying to production:

- [ ] SSL/TLS configured and tested
- [ ] Authentication and authorization configured
- [ ] Monitoring and alerting set up
- [ ] Backup procedures tested
- [ ] High availability configured (if required)
- [ ] Performance testing completed
- [ ] Security audit completed
- [ ] Disaster recovery plan documented
- [ ] Team trained on operations procedures

### Integration Testing

Validate the RabbitMQ deployment with the Clinic platform:

1. **Connection Test**: Verify all microservices can connect
2. **Message Flow Test**: Publish and consume test messages
3. **Failover Test**: Test behavior when RabbitMQ is unavailable
4. **Load Test**: Verify performance under expected load
5. **Security Test**: Verify authentication and authorization

```bash
# Simple integration test script
#!/bin/bash
echo "Testing Clinic platform integration with RabbitMQ..."

# Test 1: Connection
if rabbitmqctl ping > /dev/null; then
    echo "✓ RabbitMQ is responding"
else
    echo "✗ RabbitMQ connection failed"
    exit 1
fi

# Test 2: Virtual host exists
if rabbitmqctl list_vhosts | grep -q "clinic"; then
    echo "✓ Clinic virtual host exists"
else
    echo "✗ Clinic virtual host missing"
    exit 1
fi

# Test 3: Required exchanges exist
for exchange in "clinic.patients" "clinic.appointments" "clinic.prescriptions"; do
    if rabbitmqctl list_exchanges -p clinic | grep -q "$exchange"; then
        echo "✓ Exchange $exchange exists"
    else
        echo "✗ Exchange $exchange missing"
        exit 1
    fi
done

echo "All integration tests passed!"
```

This completes the comprehensive RabbitMQ deployment guide for the Clinic platform. The instructions provide multiple deployment options, detailed configuration guidance, and production-ready considerations for a healthcare message brokering system.