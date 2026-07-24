# ADR-012: Event Bus Abstraction over Direct BullMQ Usage

**Status:** Accepted  
**Date:** 2025-07-23  
**Deciders:** Architecture Team

---

## Context

GLU's business logic must trigger asynchronous side effects: when a book is published, the search index must be updated, notifications must be sent to followers, and analytics must be recorded. The naive implementation is direct function calls between services. The queue-aware implementation calls BullMQ directly from business logic.

Both approaches couple business logic to infrastructure. We need a better boundary.

---

## Decision

**All cross-module side effects are published as typed domain events through `IEventBus`.** Business services never import BullMQ, Redis, or any transport directly.

---

## Problem with Direct Coupling

```typescript
// BAD — BooksService knows about queues, notifications, search, analytics
class BooksService {
  constructor(
    private readonly bullMQ: Queue,
    private readonly notificationsService: NotificationsService,
    private readonly searchService: SearchService,
    private readonly analyticsService: AnalyticsService,
  ) {}

  async publishBook(id: string) {
    const book = await this.repo.findById(id);
    await this.repo.update(id, { isPublished: true });

    // Book service now knows about 4 other concerns — wrong
    await this.bullMQ.add('notify-followers', { bookId: id });
    await this.bullMQ.add('index-book', { bookId: id });
    await this.analyticsService.recordPublish(id);
    await this.searchService.indexBook(book);
  }
}
```

This creates a fan-out dependency. Every new side effect adds an import to `BooksService`. Testing requires mocking 4+ services. Extracting `BooksModule` to a microservice requires moving all those dependencies.

---

## Event-Driven Solution

```typescript
// GOOD — BooksService knows only about its own domain
class BooksService {
  constructor(
    private readonly repo: BooksRepository,
    private readonly eventBus: IEventBus,
  ) {}

  async publishBook(id: string) {
    const book = await this.repo.findById(id);
    await this.repo.update(id, { isPublished: true });

    // One publish. Subscribers handle the rest.
    await this.eventBus.publish(new BookPublishedEvent({
      bookId: book.id,
      authorId: book.authorId,
      title: book.title,
      categoryId: book.categoryId,
    }));
  }
}

// Each module subscribes independently:
// NotificationsModule subscribes to BookPublishedEvent → sends follower notifications
// SearchModule subscribes to BookPublishedEvent → indexes the book
// AnalyticsModule subscribes to BookPublishedEvent → records the publish event
// KnowledgeEngineModule subscribes to BookPublishedEvent → starts content ingestion
```

Adding a new side effect = add a new subscriber. Zero changes to `BooksService`.

---

## IEventBus Interface

```typescript
interface DomainEvent<T = unknown> {
  readonly eventName: string;
  readonly occurredAt: Date;
  readonly payload: T;
}

interface IEventBus {
  publish<T>(event: DomainEvent<T>): Promise<void>;
  publishBatch<T>(events: DomainEvent<T>[]): Promise<void>;
}

// BullMQ implementation:
class BullMQEventBus implements IEventBus {
  async publish<T>(event: DomainEvent<T>): Promise<void> {
    const queue = this.getQueueForEvent(event.eventName);
    await queue.add(event.eventName, event.payload);
  }
}

// In-memory implementation for tests:
class InMemoryEventBus implements IEventBus {
  public readonly published: DomainEvent[] = [];
  async publish<T>(event: DomainEvent<T>): Promise<void> {
    this.published.push(event);
  }
}
```

---

## Domain Event Catalogue (MVP)

| Event | Publisher | Subscribers |
|---|---|---|
| `UserRegisteredEvent` | AuthModule | NotificationsModule (welcome email) |
| `UserFollowedEvent` | UsersModule | NotificationsModule |
| `BookPublishedEvent` | BooksModule | SearchModule, NotificationsModule, AnalyticsModule, KnowledgeEngineModule |
| `BookUpdatedEvent` | BooksModule | SearchModule, KnowledgeEngineModule |
| `ChapterPublishedEvent` | ChaptersModule | NotificationsModule, SearchModule |
| `ReviewCreatedEvent` | ReviewsModule | ModerationModule, NotificationsModule |
| `CommentCreatedEvent` | CommentsModule | ModerationModule, NotificationsModule |
| `PaymentSucceededEvent` | PaymentsModule | SubscriptionsModule, NotificationsModule, AnalyticsModule |
| `ReadingSessionEndedEvent` | LibraryModule | AnalyticsModule, ChallengesModule |
| `AchievementEarnedEvent` | ChallengesModule | NotificationsModule |
| `ContentFlaggedEvent` | ModerationModule | AdminModule |
| `KnowledgeItemIngestedEvent` | KnowledgeEngineModule | AnalyticsModule |

---

## Pros

- Business logic is decoupled from side effects
- Adding new side effects never touches the publishing module
- Testing: use `InMemoryEventBus`, assert on `bus.published` array
- Transport is swappable (BullMQ → Kafka) without any service changes
- Clear audit trail — every domain event is a discrete, named occurrence
- Natural microservice boundary — event bus becomes the transport between services

## Cons

- Side effects are harder to trace (not obvious from `BooksService` alone)
- Eventual consistency: side effects happen asynchronously, not in the same transaction
- Dead letter handling required for failed event handlers

---

## Consequences

- `IEventBus` is the only event-related import in business services
- `BullMQEventBus` is the production implementation
- `InMemoryEventBus` is used in all unit and integration tests
- All domain events are defined in `src/event-bus/events/` as typed classes
- Event handlers (subscribers) live in the modules that own the side effect
- Failed events go to dead letter queue — alerting on dead letter growth
