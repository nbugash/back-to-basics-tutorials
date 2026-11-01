# Chapter 1: The Domain Core & Inversion of Control
### Milestone 1: Zero-Dependency Core Entities and Port Contracts

In this chapter, we create the pure domain layer and the ports. In Java, this is equivalent to creating your entity packages and interfaces without importing Spring, Hibernate, or Jackson.

---

## 1. Domain Entities: Structs vs. Classes

In Java, an entity is a `class` with private fields, getters, setters, and perhaps an `@Entity` annotation. In Rust, data structure and behavior are decoupled:

- **`struct`**: Holds only state/fields.
- **`impl` block**: Implements methods for that struct.
- **Derive macros**: Automate standard contracts like `Clone`, `Debug`, and `PartialEq` (similar to Lombok `@Data` or `@Value`).

### `src/domain/model.rs`
```rust
use std::fmt;

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum TaskStatus {
    Pending,
    InProgress,
    Completed,
}

impl fmt::Display for TaskStatus {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            TaskStatus::Pending => write!(f, "PENDING"),
            TaskStatus::InProgress => write!(f, "IN_PROGRESS"),
            TaskStatus::Completed => write!(f, "COMPLETED"),
        }
    }
}

/// The Core Task Entity (No ORM annotations, no database types)
#[derive(Debug, Clone, PartialEq)]
pub struct Task {
    pub id: String,
    pub title: String,
    pub description: Option<String>, // Java Optional<String>
    pub status: TaskStatus,
}

impl Task {
    /// Factory constructor (Java: public static Task create(...))
    pub fn new(id: String, title: String, description: Option<String>) -> Result<Self, String> {
        if title.trim().is_empty() {
            return Err("Task title cannot be empty".to_string());
        }
        Ok(Self {
            id,
            title,
            description,
            status: TaskStatus::Pending,
        })
    }

    /// Business method: marks task completed
    pub fn complete(&mut self) {
        self.status = TaskStatus::Completed;
    }
}
```

> **Java Comparison Note:** Notice `Option<String>` instead of `null` or `java.util.Optional<String>`. In Rust, `null` does not exist. Values are either `Some(value)` or `None`, checked strictly at compile time.

---

## 2. Idiomatic Domain Errors: `Result` vs. Exceptions

Java relies heavily on `RuntimeException` and exception inheritance trees. Rust handles errors as explicit return values via `Result<T, E>`.

### `src/domain/error.rs`
```rust
use std::fmt;

#[derive(Debug)]
pub enum DomainError {
    Validation(String),
    NotFound(String),
    Infrastructure(String),
}

impl fmt::Display for DomainError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            DomainError::Validation(msg) => write!(f, "Validation failure: {msg}"),
            DomainError::NotFound(id) => write!(f, "Resource with ID '{id}' was not found"),
            DomainError::Infrastructure(err) => write!(f, "Infrastructure error: {err}"),
        }
    }
}

impl std::error::Error for DomainError {}
```

### `src/domain/mod.rs`
```rust
pub mod model;
pub mod error;
```

---

## 3. Defining Ports: Traits vs. Java Interfaces

In Hexagonal Architecture:
1. **Inbound Ports**: Define what our core application can do (Use Cases).
2. **Outbound Ports**: Define the external capabilities required by the domain (Repositories/Gateways).

### Outbound Port (Repository Interface)
### `src/ports/outbound.rs`
```rust
use async_trait::async_trait;
use crate::domain::model::Task;
use crate::domain::error::DomainError;

/// Equivalent to: public interface TaskRepository
/// We use #[async_trait] because native async functions in object-safe traits
/// require uniform heap-allocated return futures.
#[async_trait]
pub trait TaskRepository: Send + Sync {
    async fn find_by_id(&self, id: &str) -> Result<Option<Task>, DomainError>;
    async fn save(&self, task: &Task) -> Result<(), DomainError>;
    async fn list_all(&self) -> Result<Vec<Task>, DomainError>;
}
```

> **Senior Java Insight (`Send + Sync`):** In Java, any object can generally be passed to any thread. In Rust, the compiler prevents race conditions. `Send` tells the compiler "this object can cross thread boundaries." `Sync` means "references to this object can be safely shared across threads."

### Inbound Port (Use Case Interface) & Service Implementation
### `src/ports/inbound.rs`
```rust
use std::sync::Arc;
use async_trait::async_trait;
use crate::domain::model::Task;
use crate::domain::error::DomainError;
use crate::ports::outbound::TaskRepository;

#[async_trait]
pub trait TaskUseCase: Send + Sync {
    async fn create_task(&self, title: String, description: Option<String>) -> Result<Task, DomainError>;
    async fn get_task(&self, id: &str) -> Result<Task, DomainError>;
    async fn complete_task(&self, id: &str) -> Result<Task, DomainError>;
    async fn list_tasks(&self) -> Result<Vec<Task>, DomainError>;
}

/// Use Case Implementation (Java: @Service public class TaskServiceImpl implements TaskUseCase)
pub struct TaskService {
    // In Java: private final TaskRepository repo; (Autowired via constructor)
    repo: Arc<dyn TaskRepository>,
}

impl TaskService {
    pub fn new(repo: Arc<dyn TaskRepository>) -> Self {
        Self { repo }
    }
}

#[async_trait]
impl TaskUseCase for TaskService {
    async fn create_task(&self, title: String, description: Option<String>) -> Result<Task, DomainError> {
        let id = uuid::Uuid::new_v4().to_string();
        let task = Task::new(id, title, description)
            .map_err(DomainError::Validation)?;

        self.repo.save(&task).await?;
        Ok(task)
    }

    async fn get_task(&self, id: &str) -> Result<Task, DomainError> {
        self.repo.find_by_id(id).await?
            .ok_or_else(|| DomainError::NotFound(id.to_string()))
    }

    async fn complete_task(&self, id: &str) -> Result<Task, DomainError> {
        let mut task = self.get_task(id).await?;
        task.complete();
        self.repo.save(&task).await?;
        Ok(task)
    }

    async fn list_tasks(&self) -> Result<Vec<Task>, DomainError> {
        self.repo.list_all().await
    }
}
```

### `src/ports/mod.rs`
```rust
pub mod inbound;
pub mod outbound;
```

---

## 4. Milestone 1 Verification: Isolated Unit Tests

To verify that the domain and port logic can be thoroughly tested using an in-memory mock (just like Mockito in Java), add this test to `src/ports/inbound.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::sync::Mutex;

    // In-memory Mock Repository (Java: @MockBean TaskRepository repo)
    struct MockTaskRepository {
        storage: Mutex<Vec<Task>>,
    }

    impl MockTaskRepository {
        fn new() -> Self {
            Self { storage: Mutex::new(Vec::new()) }
        }
    }

    #[async_trait]
    impl TaskRepository for MockTaskRepository {
        async fn find_by_id(&self, id: &str) -> Result<Option<Task>, DomainError> {
            let list = self.storage.lock().unwrap();
            Ok(list.iter().find(|t| t.id == id).cloned())
        }

        async fn save(&self, task: &Task) -> Result<(), DomainError> {
            let mut list = self.storage.lock().unwrap();
            if let Some(pos) = list.iter().position(|t| t.id == task.id) {
                list[pos] = task.clone();
            } else {
                list.push(task.clone());
            }
            Ok(())
        }

        async fn list_all(&self) -> Result<Vec<Task>, DomainError> {
            let list = self.storage.lock().unwrap();
            Ok(list.clone())
        }
    }

    #[tokio::test]
    async fn test_create_and_complete_task() {
        let mock_repo = Arc::new(MockTaskRepository::new());
        let service = TaskService::new(mock_repo);

        let created = service.create_task("Write Tests".to_string(), None).await.unwrap();
        assert_eq!(created.title, "Write Tests");
        assert_eq!(created.status.to_string(), "PENDING");

        let completed = service.complete_task(&created.id).await.unwrap();
        assert_eq!(completed.status.to_string(), "COMPLETED");
    }
}
```

Run test suite:
```bash
cargo test ports::inbound::tests
```
You have achieved **Milestone 1**: A pristine domain business core shielded behind clean abstract contracts.