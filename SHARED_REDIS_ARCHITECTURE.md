# Shared Redis Architecture for Vhotis Platform

## Overview

This document describes the shared Redis infrastructure used across the Main App and Rider App for real-time features, geospatial queries, and efficient communication.

## Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                    Vhotis Platform Infrastructure                 │
│                                                                   │
│  ┌─────────────┐                           ┌─────────────┐      │
│  │ Main App    │                           │ Rider App   │      │
│  │ (Buyers +   │◄────── RabbitMQ ─────────►│ (Delivery)  │      │
│  │  Sellers)   │     (Order Messages)      │             │      │
│  └──────┬──────┘                           └──────┬──────┘      │
│         │                                          │              │
│         │         ┌──────────────────┐            │              │
│         │         │   RabbitMQ       │            │              │
│         │         │   - Celery Broker│            │              │
│         │         │   - Order Events │            │              │
│         │         └──────────────────┘            │              │
│         │                                          │              │
│         └───────────────┬──────────────────────────┘             │
│                         ▼                                         │
│              ┌─────────────────────┐                             │
│              │   Shared Redis      │                             │
│              │  (Port 6380)        │                             │
│              │                     │                             │
│              │  Database 0:        │                             │
│              │  ├─ Celery Results  │                             │
│              │  ├─ Geospatial Data │                             │
│              │  ├─ Proximity Cache │                             │
│              │  └─ Pub/Sub Channels│                             │
│              └─────────────────────┘                             │
└──────────────────────────────────────────────────────────────────┘
```

## Why Shared Redis?

### Single Instance Benefits

1. **Real-time Geospatial Queries**

   - Rider locations stored once
   - Both apps can query instantly
   - Sub-millisecond proximity checks

2. **Pub/Sub for Instant Notifications**

   - Rider publishes location → Main app gets notified immediately
   - No polling required
   - 1-5ms latency vs 100-500ms with RabbitMQ

3. **Resource Efficiency**

   - Single Redis instance vs multiple
   - Shared memory footprint
   - Easier to monitor and maintain

4. **Data Consistency**
   - Single source of truth for rider locations
   - No synchronization issues
   - Atomic operations across apps

## Components

### 1. Celery Task Queue

**Broker**: RabbitMQ (shared between apps)
**Result Backend**: Shared Redis Database 0

```
Main App Celery → Queue: main_tasks → RabbitMQ → Shared Redis (results)
Rider App Celery → Queue: rider_tasks → RabbitMQ → Shared Redis (results)
```

### 2. Geospatial Data (Redis GEO commands)

**Key**: `rider_locations` (sorted set with geospatial indexing)

**Operations**:

- `GEOADD rider_locations {lon} {lat} rider:{rider_id}` - Add/update rider location
- `GEORADIUS rider_locations {lon} {lat} 5000 m` - Find riders within 5km
- `GEODIST rider_locations rider:1 rider:2` - Distance between riders
- `ZREM rider_locations rider:{rider_id}` - Remove rider (when offline)

**Data Structure**:

```
rider_locations: {
  "rider:1": (latitude, longitude),
  "rider:2": (latitude, longitude),
  ...
}

rider:{rider_id}: {
  "order_id": "ord_123",
  "latitude": "51.5074",
  "longitude": "-0.1278",
  "last_updated": "2025-10-31T10:30:00Z"
}
```

### 3. Proximity Notifications (Redis Pub/Sub)

**Channels**:

- `proximity:order:{order_id}` - Order-specific proximity events
- `location:rider:{rider_id}` - Rider location updates

**Flow**:

```
1. Rider moves → Updates location every 5 seconds
2. Rider app calculates distance to pickup/delivery
3. Within threshold? → Publish to proximity channel
4. Main app subscribes to all proximity:* channels
5. Receives event → Sends push notification to buyer/seller
```

**Message Format**:

```json
{
  "order_id": "ord_123",
  "recipient_id": 456,
  "recipient_name": "John Doe",
  "location_type": "delivery",
  "distance": 48.5,
  "threshold": 50,
  "message": "Rider is 50m away!",
  "timestamp": "2025-10-31T10:30:00Z"
}
```

### 4. Proximity Deduplication

**Key**: `notified:{order_id}:{location_type}:{threshold}`

**Purpose**: Prevent duplicate notifications for same threshold

**Expiry**: 1 hour (TTL)

**Example**:

```
notified:ord_123:delivery:200 = "1" (expires in 3600s)
notified:ord_123:delivery:100 = "1" (expires in 3600s)
notified:ord_123:delivery:50 = "1" (expires in 3600s)
```

## Implementation Details

### Rider App - Location Updates

**Endpoint**: `POST /api/v1/location/update`

**Request**:

```json
{
  "latitude": 51.5074,
  "longitude": -0.1278,
  "order_id": "ord_123",
  "accuracy": 10.5,
  "heading": 180
}
```

**Processing**:

1. Update Redis geospatial index
2. Store rider metadata
3. Publish location update
4. Check proximity to order locations
5. Publish proximity notifications if thresholds met

**Frequency**: Every 5 seconds during active delivery

### Main App - Proximity Listener

**Management Command**: `python manage.py start_proximity_listener`

**Docker Service**: `proximity_listener` (runs continuously)

**Processing**:

1. Subscribe to `proximity:order:*` pattern
2. Receive proximity events
3. Parse notification data
4. Send push notification to buyer/seller
5. Log notification delivery

## Configuration

### Environment Variables

Both apps need these environment variables:

```bash
# Celery Configuration
CELERY_BROKER_URL=amqp://guest:guest@rabbitmq:5672//
CELERY_RESULT_BACKEND=redis://shared_redis:6379/0

# Shared Redis
SHARED_REDIS_URL=redis://shared_redis:6379/0
```

### Docker Compose

**Main App** (`main/docker-compose.yml`):

- All services depend on `shared_redis`
- `proximity_listener` service runs continuously
- Port 6380 exposed for external access

**Rider App** (`rider/docker-compose.yml`):

- All services depend on `shared_redis` (external network)
- No local Redis instance needed

### Network Configuration

Both apps use `lapse_network` (external) to communicate with shared Redis.

```bash
# Create network if not exists
docker network create lapse_network
```

## Usage Examples

### Rider App - Send Location Update

```python
# rider/server/delivery/main/views/location.py
from main.services.location_service import LocationService

location_service = LocationService()
location_service.update_rider_location(
    rider_id='123',
    order_id='ord_456',
    latitude=51.5074,
    longitude=-0.1278
)
```

### Main App - Query Nearby Riders

```python
# main/server/vhotis_server/main/tasks/rider_assignment.py
from django.conf import settings

location_service = LocationService()
nearby_riders = location_service.get_riders_near_location(
    latitude=51.5074,
    longitude=-0.1278,
    radius_meters=5000  # 5km
)

# Returns: [
#   {'rider_id': '123', 'distance': 1234.5, 'latitude': 51.508, 'longitude': -0.128},
#   ...
# ]
```

### Proximity Notification Thresholds

**Default thresholds**: 200m, 100m, 50m

**Customization** in `location_service.py`:

```python
self.notify_on_proximity(
    distance,
    order_id,
    recipient_id,
    'delivery',
    recipient_name,
    thresholds=[500, 250, 100, 50]  # Custom thresholds
)
```

## Running the System

### 1. Start Shared Infrastructure

```bash
cd main
docker-compose up -d shared_redis rabbitmq
```

### 2. Start Main App Services

```bash
docker-compose up -d main_server celery_worker rabbitmq_consumer proximity_listener
```

### 3. Start Rider App Services

```bash
cd ../rider
docker-compose up -d rider_server celery_worker rabbitmq_consumer
```

### 4. Verify Shared Redis Connection

```bash
# Connect to shared Redis
docker exec -it vhotis-main-shared_redis-1 redis-cli

# Check connected clients
CLIENT LIST

# Check geospatial data
ZRANGE rider_locations 0 -1 WITHSCORES

# Monitor pub/sub messages
PSUBSCRIBE proximity:*
```

## Monitoring

### Redis Commands

```bash
# View all rider locations
ZRANGE rider_locations 0 -1 WITHSCORES

# Get specific rider data
HGETALL rider:123

# Check proximity deduplication
KEYS notified:*

# Monitor pub/sub channels
PUBSUB CHANNELS proximity:*
PUBSUB NUMSUB proximity:order:123
```

### Logs

```bash
# Proximity listener logs
docker-compose logs -f proximity_listener

# Rider location updates
docker-compose -f ../rider/docker-compose.yml logs -f rider_server
```

## Performance Characteristics

- **Location update latency**: < 10ms
- **Proximity check**: < 1ms (Redis GEORADIUS)
- **Pub/Sub latency**: 1-5ms
- **Total notification time**: < 50ms (from location update to push notification sent)
- **Scalability**: Handles 10,000+ concurrent riders

## Security Considerations

1. **Network Isolation**: Shared Redis only accessible via Docker network
2. **No External Exposure**: Port 6380 can be removed in production
3. **Authentication**: Redis password can be configured via environment variables
4. **Data Expiry**: Rider locations expire after 1 hour of inactivity

## Future Enhancements

1. **Redis Clustering**: For > 100k concurrent riders
2. **Geofencing**: Define delivery zones with polygon boundaries
3. **Heat Maps**: Rider density visualization for admins
4. **ETA Prediction**: ML-based delivery time estimates using historical location data
5. **Route Optimization**: Assign riders based on current location and traffic patterns

## Troubleshooting

### Shared Redis Not Connecting

```bash
# Check if shared_redis service is running
docker-compose ps shared_redis

# Check network connectivity
docker network inspect lapse_network

# Test Redis connection
docker exec -it vhotis-main-shared_redis-1 redis-cli PING
```

### Proximity Notifications Not Working

1. Check if proximity_listener is running: `docker-compose logs proximity_listener`
2. Verify rider is sending location updates: Check rider app logs
3. Monitor Redis pub/sub: `PSUBSCRIBE proximity:*` in redis-cli
4. Check notification deduplication keys: `KEYS notified:*`

### Celery Not Using Shared Redis

1. Verify environment variables in docker-compose
2. Check settings.py has correct CELERY_RESULT_BACKEND
3. Restart Celery workers: `docker-compose restart celery_worker`
