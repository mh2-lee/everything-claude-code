---
description: Restate requirements, assess risks, and create step-by-step implementation plan. WAIT for user CONFIRM before touching any code.
---

# Plan Command

This command invokes the **planner** agent to create a comprehensive implementation plan before writing any code.

## What This Command Does

1. **Restate Requirements** - Clarify what needs to be built
2. **Identify Risks** - Surface potential issues and blockers
3. **Create Step Plan** - Break down implementation into phases
4. **Wait for Confirmation** - MUST receive user approval before proceeding

## When to Use

Use `/plan` when:
- Starting a new feature
- Making significant architectural changes
- Working on complex refactoring
- Multiple files/components will be affected
- Requirements are unclear or ambiguous

## How It Works

The planner agent will:

1. **Analyze the request** and restate requirements in clear terms
2. **Break down into phases** with specific, actionable steps
3. **Identify dependencies** between components
4. **Assess risks** and potential blockers
5. **Estimate complexity** (High/Medium/Low)
6. **Present the plan** and WAIT for your explicit confirmation

## Example Usage

```
User: /plan I need to add real-time notifications when orders are completed

Agent (planner):
# Implementation Plan: Real-Time Order Completion Notifications

## Requirements Restatement
- Send notifications to users when their orders are completed
- Support multiple notification channels (in-app, email, webhook)
- Ensure notifications are delivered reliably
- Include order details and status in notification

## Implementation Phases

### Phase 1: Database Schema
- Add notifications table: id, user_id, order_id, type, status, created_at
- Add user_notification_preferences table for channel preferences
- Create Flyway migration scripts
- Add indexes on user_id and order_id for performance

### Phase 2: Notification Service
- Create NotificationService in notification package
- Implement notification queue using Spring AMQP/RabbitMQ
- Add retry logic with @Retryable annotation
- Create notification templates

### Phase 3: Event Integration
- Create OrderCompletedEvent class
- Publish event from OrderService when status changes to COMPLETED
- Create NotificationEventListener with @EventListener
- Enqueue notifications for each relevant user

### Phase 4: REST API & WebSocket
- Create NotificationController for REST endpoints
- Implement WebSocket with Spring WebSocket/STOMP
- Add real-time push notifications
- Create notification preferences endpoint

## Dependencies
- RabbitMQ (for message queue)
- Spring Mail (for email notifications)
- Spring WebSocket (for real-time updates)

## Risks
- HIGH: Email deliverability (SPF/DKIM configuration required)
- MEDIUM: Performance with high volume orders
- MEDIUM: WebSocket connection management at scale
- LOW: Message queue reliability

## Estimated Complexity: MEDIUM

**WAITING FOR CONFIRMATION**: Proceed with this plan? (yes/no/modify)
```

## Important Notes

**CRITICAL**: The planner agent will **NOT** write any code until you explicitly confirm the plan with "yes" or "proceed" or similar affirmative response.

If you want changes, respond with:
- "modify: [your changes]"
- "different approach: [alternative]"
- "skip phase 2 and do phase 3 first"

## Integration with Other Commands

After planning:
- Use `/tdd` to implement with test-driven development
- Use `/build-fix` if build errors occur
- Use `/code-review` to review completed implementation

## Related Agents

This command invokes the `planner` agent located at:
`~/.claude/agents/planner.md`
