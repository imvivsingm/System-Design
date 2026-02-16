Title:
Efficient Multiplexed Real-Time Market Data Distribution Service for Multiple End Users from a Single LSEG WebSocket Feed

Objective:
Build a scalable, low-latency, and cost-efficient real-time pricing distribution system that allows N concurrent end-users (potentially hundreds or thousands) to view live market data (prices, quotes, trades, etc.) for various financial instruments (RICs), while maintaining exactly one active subscription per RIC on the upstream LSEG WebSocket connection — regardless of how many users are interested in the same instrument.

Core Constraint (Non-Negotiable):

The system must never create more than one subscription/request for the same RIC (e.g. "AAPL.O", "RELIANCE.NS", "GBPJPY=") to the LSEG WebSocket feed, even if 1, 10, 500 or more users are simultaneously viewing or monitoring that instrument.
Violating this constraint would result in unnecessary bandwidth consumption, increased latency risk, higher operational cost, and potential violation of LSEG usage agreements or throttling limits.
Functional Requirements:

Single Upstream Connection
Exactly one persistent WebSocket connection to the LSEG real-time market data feed (either raw WebSocket API or LSEG Data Platform / EMA).
The connection must handle login, authentication, heartbeat (ping/pong), reconnection, and token refresh automatically.
Subscription Multiplexing / Reference Counting
When any end-user requests to view data for a RIC:
If no other user is currently subscribed to that RIC → create one subscription on the LSEG connection.
If the RIC is already subscribed (at least one other user is watching) → do not create a new upstream subscription.
Track the number of active consumers per RIC (reference count).
When the last user stops watching a RIC (explicit unsubscribe or client disconnect) → remove the upstream subscription from LSEG (send close/unsubscribe message).
Real-Time Fan-Out / Broadcast
Every Refresh, Update, Correction, or Status message received from LSEG for a RIC must be immediately forwarded to all currently connected end-users who are subscribed to that RIC.
Delivery should be low-latency (ideally sub-10 ms from receipt to client delivery under normal load).
End-User Interface
End-users connect to the system via WebSocket (preferred) or alternative transport (SSE, long-polling fallback).
Users can dynamically subscribe/unsubscribe to any number of RICs during their session.
Users receive only the data for the RICs they are actively watching.
Correctness Guarantees
No duplicate upstream subscriptions for the same RIC.
No data loss: every user who is subscribed must receive every update.
Clean cleanup: when a client disconnects abnormally, all of its subscriptions are released (no zombie subscriptions).
Non-Functional Requirements (Production Expectations):

Performance
Handle ≥ 1,000 concurrent WebSocket client connections per server instance
Support ≥ 5,000–10,000 active RIC subscriptions across all users (limited by LSEG connection capacity, not by the application)
Message processing latency < 5 ms (p99) from LSEG receipt to client dispatch under normal load
Reliability & Resilience
Automatic reconnection to LSEG on disconnect / network failure with exponential backoff + jitter
Graceful degradation when LSEG connection is down (queue or drop non-critical updates, show status to users)
No memory leak on long-running subscriptions or frequent connect/disconnect cycles
Observability
Metrics: active upstream subscriptions, total downstream clients, messages/sec per RIC, latency histograms, reconnect attempts
Logging: subscription lifecycle events, errors, LSEG connection state
Health check endpoint reflecting upstream connection status
Security
Authenticate end-users (JWT, OAuth2, or session-based) before allowing subscription
Prevent users from subscribing to arbitrary or restricted RICs (if applicable)
Scalability Path
Design must allow horizontal scaling (multiple application instances) without creating duplicate upstream subscriptions
Support future transition to a dedicated “feed gateway” + pub/sub bus (Kafka, Redis Pub/Sub, etc.) pattern
In Scope (for initial version):

Real-time market price domain (MarketPrice / Level 1)
JSON-based LSEG WebSocket API (or EMA Java if switching)
Single JVM deployment (Spring Boot)
In-memory subscription state management
Out of Scope (for v1):

Historical data / snapshot requests
Order book (Level 2) data
Multi-region / geo-redundant LSEG connections
Advanced conflation / update filtering
Persistent subscription state across restarts
Success Criteria:
When 50 users connect and 10 of them subscribe to "AAPL.O", the system creates exactly 1 subscription request to LSEG for AAPL.O, and all 10 users receive every subsequent update — with only one upstream connection active at all times.

This is the crisp, unambiguous problem statement we would use internally at a high-frequency trading firm or market-data vendor to align engineering, product, and compliance teams before writing any code.

Does this capture your intent accurately?
If yes → next step is usually:

Detailed component design & data flow diagram
Critical state management decisions (in-memory vs Redis)
Error & reconnection handling strategy
First prototype code outline
Let me know what you want to drill into next.
