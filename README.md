In a huge corporation, a senior backend developer gets a JIRA ticket:

> "Expose user's `loyaltyPoints` in the `/user/profile` API response.
> Estimate: 1 story point."

He squints.

"1 point? This is clearly a *platform-level concern*."

Sprint 1: Discovery
He opens the code.

There is no `/user/profile`. There is:

* `/v1/user/profile`
* `/v1/user/profile2`
* `/v2/user/profile`
* `/gateway/user/profile-aggregated`
* `/edge/user/profile-lightweight`

And a microservice called `user-profile-orchestrator` that just forwards requests and logs "TODO".

He finds the loyalty points stored in another service:

* `loyalty-core`
* `loyalty-read`
* `loyalty-write`
* `loyalty-legacy`
* `loyalty-bridge` (nobody knows what it bridges)

Each has its own database, plus one undocumented Redis cluster that will segfault if you look at it too hard.

He writes his first comment on the ticket:

> "We need to align on the *source of truth* for loyalty points."

Sprint 2: Architecture
He calls a meeting with 14 people titled:

> "Loyalty Points Data Ownership and Real Time Access Strategy"

Slides:

* "Problem": Mobile app wants to show how many points user has.
* "Solution": Global event driven architecture with exactly once semantics, CQRS, and a read optimized profile view.

Someone from DevOps suggests:
"Or we could just JOIN the tables in the existing API."

He writes on the Miro board:
"Anti pattern. Tight coupling. Technical debt."
Then erases it so nobody can quote him later.

Outcome:

* New proposal: `user-profile-read-model` service.
* Tech stack: Kotlin + Spring + Kafka + Postgres + Redis + OpenAPI + gRPC for internal calls "because REST is legacy".
* Status: Not started.

Sprint 3: Implementation
He scaffolds a new service in the monorepo:

`services/user-profile-read-model`

Generates 73 files with the internal boilerplate CLI.

Actual logic so far:

```kotlin
@RestController
class HealthController {
    @GetMapping("/health")
    fun health() = mapOf("status" to "OK")
}
```

He pings the team:

"Service is up in dev."

Product Manager: "So can I see loyaltyPoints in the response now?"

Backend dev: "Not yet, we are still discussing the contract."

Two weeks pass. They reach consensus on naming:

* `loyaltyPoints`
* not `points`
* not `loyalty_points`
* definitely not `lp_balance` ("too DBA")

They write an internal RFC to document this historic decision.

Sprint 4: Events
Now he has to "stream" the data.

He creates 5 Kafka topics:

* `user-loyalty-points-changed`
* `user-loyalty-points-compacted`
* `user-loyalty-points-dead-letter`
* `user-loyalty-points-replay-requested`
* `user-loyalty-points-replay-completed`

He wires a consumer:

```kotlin
@KafkaListener(topics = ["user-loyalty-points-changed"])
fun handleLoyaltyEvent(event: LoyaltyPointsChanged) {
    // TODO: implement
}
```

He commits with message:
`feat: skeleton for loyalty points event processing`

PR reviewers:

* "Can we make this more generic, in case in future we have different types of loyalty?"
* "Can we abstract the Kafka layer so we can swap it out later?"
* "Can we add tracing, metrics, structured logging and correlation IDs?"

They now have a full observability stack for an empty function.

Sprint 5: Data model
He designs the table:

```sql
CREATE TABLE user_profile_read_model (
  user_id UUID PRIMARY KEY,
  email TEXT NOT NULL,
  name TEXT NOT NULL,
  loyalty_points BIGINT NOT NULL DEFAULT 0,
  updated_at TIMESTAMP NOT NULL DEFAULT now()
);
```

DBA jumps in:

* "We do not use BIGINT for points. What if the business changes the definition of points?"

They agree to change it to `NUMERIC(20, 4)` because "you never know".

He writes the projector:

```kotlin
fun handleLoyaltyEvent(event: LoyaltyPointsChanged) {
    jdbcTemplate.update(
        """
        INSERT INTO user_profile_read_model (user_id, loyalty_points, updated_at)
        VALUES (?, ?, now())
        ON CONFLICT (user_id) DO UPDATE
        SET loyalty_points = EXCLUDED.loyalty_points,
            updated_at = now()
        """.trimIndent(),
        event.userId,
        event.points
    )
}
```

Now the data actually appears in the database.

Still not in the API.

Sprint 6: API Gateway
He opens the code for the "gateway" service that exposes `/user/profile`.

The file is named `UserProfileController.java`, last touched 2017.

Inside:

* 3 different HTTP clients
* 1 homegrown circuit breaker
* 0 tests
* 17 `// TODO` comments
* manual JSON building with `StringBuilder`

He thinks for 10 seconds.

Then does the correct thing: closes the file, writes a new gateway in Kotlin.

"Much faster to rewrite than to understand," he explains in his design doc.

The new gateway:

* Uses WebFlux
* Streams responses
* Has OpenTelemetry tracing
* Supports feature flags per field
* Has a configuration that lets you turn off loyalty points per region in case "Germany complains"

Sprint 7: Edge cases
Security shows up:

"Can users guess other users' loyaltyPoints?"

He explains that you need a JWT, a user ID, and a VPN into production to even hit the endpoint.

Legal shows up:

"If loyalty points can be interpreted as currency, do we now become some kind of bank?"

They add footnotes to the API docs.

Compliance shows up:

"In some countries, loyalty programs are considered gambling."

They add an environment variable:

```yaml
DISABLE_LOYALTY_POINTS_IN: ["IT", "DE", "CH", "FR", "PROD"]
```

Nobody notices "PROD" is on that list.

Sprint 8: QA
QA tests it in dev:

* Login as test user, calls `/user/profile`
* Sees:

```json
{
  "id": "123",
  "name": "Test User",
  "loyaltyPoints": 420
}
```

"Looks good."

They write 20 more test cases for:

* user without loyalty
* user with negative points (legacy bug)
* user with 2 accounts
* user from forbidden country

All pass.

They mark the ticket "Ready for Release".

Sprint 9: Release
They deploy to staging, then prod, behind a flag:

```yaml
FEATURE_FLAGS:
  USER_PROFILE_READ_MODEL: false
  USER_PROFILE_LOYALTY_POINTS: false
```

They create a rollout plan:

* Week 1: internal users only
* Week 2: 10% of real users
* Week 3: 50% of users
* Week 4: 100%, unless "metrics indicate issues"

Metrics to monitor:

* p95 latency of `/user/profile`
* error rate
* "user happiness", approximated by number of support tickets containing the word "points"

Two months later, the flags are still `false`.

Mobile team pings:

"Hey, still not seeing `loyaltyPoints` in production. Any updates?"

Backend dev:
"We are just validating the stability of the new architecture."

Mobile dev:
"We solved it on our side already."

Backend dev:
"...how?"

Mobile dev:
"There was an old undocumented endpoint: `/user/debug/full-profile-internal` that returns everything, including `loyalty_points`. We are using that."

Silence in the channel.

Monitoring:

* Latency doubled.
* Payload size tripled.
* That endpoint is not cached, not rate limited, and not behind auth in one of the regions.

Security opens a P1 incident.

Postmortem:
Root cause:

> "Mobile app used undocumented internal endpoint instead of official API."

Action items:

* "Backend to provide a stable, documented `loyaltyPoints` field in `/user/profile`"
* "Mobile team to stop using debug endpoints"
* "Everyone to follow the API governance process"

Backend dev opens a new JIRA:

> "Epic: Deprecate `debug/full-profile-internal` and migrate consumers to `user-profile-read-model`"

Estimate: 13 story points.

He moves the old ticket "Expose loyaltyPoints in `/user/profile`" to "Blocked" because now it depends on the new epic.

End of year performance review:

Manager:
"So, what did you ship this year?"

Backend dev:

* "I designed and implemented an event driven read model for user profiles, introduced a typed Kafka contract for loyalty events, replaced the legacy gateway with an observable, secure, feature flagged API, and established the foundation for future personalization."

Manager scrolls through dashboards.

"Huh. According to the logs, the only thing actually serving loyalty points to users in production right now is a debug endpoint from 2016 that someone wrote in pure PHP."

Pause.

Manager:
"Anyway, great job modernizing the platform. Rating: Meets expectations."

As he closes his laptop, he gets tagged in a fresh JIRA:

> "Task: Add `hasLoyalty` boolean to `/user/profile` response. 1 story point."

He opens Confluence and starts a new document:

> "Loyalty Capability Projection Flag: High Level Design"

Somewhere in the corner, an intern runs:

```sql
ALTER TABLE user_profile ADD COLUMN has_loyalty BOOLEAN DEFAULT TRUE;
```

commits:

```diff
- "loyaltyPoints": 0
+ "loyaltyPoints": 0,
+ "hasLoyalty": true
```

pushes to prod.

Then spends the rest of the day trying to get access to the Kafka UI.
