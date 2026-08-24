# Diagrama de clases de M-Team

Este diagrama representa en una única vista las entidades del dominio definidas en el alcance.

Los atributos son privados, los métodos públicos comienzan con un verbo y las asociaciones indican multiplicidad en ambos extremos. No se muestran claves foráneas porque pertenecen al modelo relacional.

```mermaid
classDiagram
direction TB

  class User {
    - id: UUID
    - firstName: String
    - lastName: String
    - documentNumber: String
    - birthDate: Date
    - email: String
    - phone: String
    - passwordHash: String
    - photoUrl: String [0..1]
    - role: UserRole
    - status: UserStatus
    - hasTemporaryPassword: Boolean
    - createdAt: DateTime
    - updatedAt: DateTime
    + updateContact(email: String, phone: String) void
    + changePassword(passwordHash: String) void
    + assignTemporaryPassword(passwordHash: String) void
    + activate() void
    + deactivate() void
    + canAuthenticate() Boolean
  }

  class MemberProfile {
    - id: UUID
    - emergencyContactName: String
    - emergencyContactPhone: String
    + updateEmergencyContact(name: String, phone: String) void
  }

  class TrainerProfile {
    - id: UUID
    - specialty: String
    - description: String
    + updateProfessionalProfile(specialty: String, description: String) void
  }

  class UserAuditLog {
    <<immutable>>
    - id: UUID
    - action: UserAuditAction
    - reason: String [0..1]
    - occurredAt: DateTime
  }

  class MembershipPrice {
    <<immutable>>
    - id: UUID
    - amount: Decimal
    - effectiveFrom: DateTime
    - createdAt: DateTime
  }

  class Payment {
    - id: UUID
    - amount: Decimal
    - method: String
    - receiptNumber: String [0..1]
    - status: PaymentStatus
    - createdAt: DateTime
    - accreditedAt: DateTime
    - expiresAt: DateTime
    - voidedAt: DateTime [0..1]
    - voidReason: String [0..1]
    + accredit(accreditedAt: DateTime) void
    + voidPayment(reason: String, voidedAt: DateTime) void
    + calculateExpiration() DateTime
    + isValidAt(date: DateTime) Boolean
  }

  class MedicalCertificate {
    - id: UUID
    - fileUrl: String
    - status: MedicalCertificateStatus
    - uploadedAt: DateTime
    - reviewedAt: DateTime [0..1]
    - reviewComment: String [0..1]
    + approve(reviewedAt: DateTime) void
    + reject(comment: String, reviewedAt: DateTime) void
    + isApproved() Boolean
  }

  class Branch {
    - id: UUID
    - name: String
    - description: String
    - imageUrl: String
    - address: String
    - openingHours: String
    - phone: String
    - latitude: Decimal [0..1]
    - longitude: Decimal [0..1]
    - isActive: Boolean
    + activate() void
    + deactivate() void
  }

  class AccessPoint {
    - id: UUID
    - name: String
    - qrToken: String
    - isActive: Boolean
    + matchesQrToken(token: String) Boolean
    + activate() void
    + deactivate() void
  }

  class AccessLog {
    <<immutable>>
    - id: UUID
    - roleAtAttempt: UserRole
    - result: AccessResult
    - denialReason: AccessDenialReason [0..1]
    - attemptedAt: DateTime
  }

  class WeeklySchedule {
    - id: UUID
    - weekStartsOn: Date
    - createdAt: DateTime
    + copyToWeek(weekStartsOn: Date) WeeklySchedule
  }

  class ScheduledClass {
    - id: UUID
    - activity: String
    - startsAt: DateTime
    + reschedule(startsAt: DateTime) void
    + assignTrainer(trainer: TrainerProfile) void
    + removeTrainer() void
    + changeBranch(branch: Branch) void
  }

  class Event {
    - id: UUID
    - title: String
    - description: String
    - startsAt: DateTime
    - location: String
    - imageUrl: String [0..1]
    - status: EventStatus
    + publish() void
    + cancel() void
    + isFinishedAt(date: DateTime) Boolean
  }

  class NewsPost {
    - id: UUID
    - title: String
    - content: String
    - imageUrl: String [0..1]
    - audience: PublicationAudience
    - status: PublicationStatus
    - publishedAt: DateTime [0..1]
    + publish(publishedAt: DateTime) void
    + updateContent(title: String, content: String) void
    + deactivate() void
  }

  class Notification {
    - id: UUID
    - title: String
    - message: String
    - type: NotificationType
    - createdAt: DateTime
    - readAt: DateTime [0..1]
    + markAsRead(readAt: DateTime) void
    + isRead() Boolean
  }

User "1" *-- "0..1" MemberProfile : owns
User "1" *-- "0..1" TrainerProfile : owns
User "1" -- "0..*" UserAuditLog : auditedUser
User "1" -- "0..*" UserAuditLog : performedBy

MemberProfile "1" -- "0..*" Payment : has
MemberProfile "1" -- "0..*" MedicalCertificate : submits
User "1" -- "0..*" MembershipPrice : creates
User "1" -- "0..*" Payment : confirms
User "0..1" -- "0..*" Payment : voids
User "0..1" -- "0..*" MedicalCertificate : reviews

Branch "1" *-- "0..*" AccessPoint : contains
User "1" -- "0..*" AccessLog : attempts
Branch "0..1" -- "0..*" AccessLog : receives
AccessPoint "0..1" -- "0..*" AccessLog : records

TrainerProfile "0..*" -- "0..*" Branch : worksAt
WeeklySchedule "1" *-- "0..*" ScheduledClass : contains
WeeklySchedule "0..1" -- "0..*" WeeklySchedule : copiedFrom
Branch "1" -- "0..*" ScheduledClass : hosts
TrainerProfile "0..1" -- "0..*" ScheduledClass : teaches

User "1" -- "0..*" Event : creates
User "1" -- "0..*" NewsPost : creates
User "1" -- "0..*" Notification : receives
```

## Enumeraciones

| Enum | Valores |
|---|---|
| `UserRole` | `MEMBER`, `TRAINER`, `ADMIN` |
| `UserStatus` | `ACTIVE`, `INACTIVE` |
| `MembershipStatus` | `CURRENT`, `EXPIRING_SOON`, `EXPIRED` |
| `PaymentStatus` | `ACCREDITED`, `VOIDED` |
| `MedicalCertificateStatus` | `PENDING`, `APPROVED`, `REJECTED` |
| `AccessResult` | `ALLOWED`, `DENIED` |
| `AccessDenialReason` | `INVALID_QR`, `INACTIVE_USER`, `INACTIVE_BRANCH`, `INACTIVE_ACCESS_POINT`, `EXPIRED_MEMBERSHIP`, `MEDICAL_CERTIFICATE_REQUIRED` |
| `UserAuditAction` | `CREATED`, `UPDATED`, `ACTIVATED`, `DEACTIVATED`, `PASSWORD_RESET` |
| `EventStatus` | `DRAFT`, `PUBLISHED`, `CANCELLED` |
| `PublicationAudience` | `ALL`, `MEMBERS`, `TRAINERS` |
| `PublicationStatus` | `DRAFT`, `PUBLISHED`, `INACTIVE` |
| `NotificationType` | `MEMBERSHIP_PRICE_CHANGED`, `MEMBERSHIP_EXPIRING`, `MEMBERSHIP_EXPIRED`, `MEDICAL_CERTIFICATE_REVIEWED`, `CLASS_CHANGED`, `EVENT_CANCELLED`, `GENERAL` |

## Restricciones y decisiones

- `MemberProfile` y `TrainerProfile` son mutuamente excluyentes. `MEMBER` requiere `MemberProfile`, `TRAINER` requiere `TrainerProfile` y `ADMIN` no requiere ninguno.
- `MembershipStatus` se calcula desde los pagos acreditados y no se almacena como atributo.
- `UserAuditLog` y `AccessLog` son registros históricos inmutables; por eso no exponen métodos modificadores.
- `Branch` y `AccessPoint` son opcionales en `AccessLog` solamente cuando el QR es inexistente o fue alterado.
- Cada pago acreditado genera 30 días de vigencia sin acumular días anteriores.
- Al anular un pago se recalculan la vigencia y el período inicial desde los pagos que permanezcan válidos.
- El período inicial para ingresar sin apto aprobado dura 20 días desde el primer pago acreditado válido.
- Un apto aprobado permanece válido sin fecha de vencimiento.
- El QR identifica un punto de acceso fijo; el usuario se identifica mediante su sesión.
- Las clases son informativas y no incluyen reservas, cupos, listas de espera ni asistencia.
- Los servicios, controladores, repositorios y DTO se documentan en el diseño de capas y no se modelan como entidades.

## Decisión pendiente

M-Team debe confirmar los medios de pago aceptados. Hasta entonces, `Payment.method` permanece como `String`.
