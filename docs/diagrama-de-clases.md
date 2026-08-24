# Diagrama de clases de M-Team

Este documento representa las clases del dominio definidas por el alcance de M-Team. Utiliza UML con atributos privados, operaciones públicas relevantes, relaciones tipadas y multiplicidades en ambos extremos.

No incluye claves foráneas, tablas, controladores, DTO ni detalles de Prisma. Esos elementos pertenecen al modelo relacional y al diseño de capas. Los nombres del código se expresan en inglés, de acuerdo con las convenciones del proyecto.

## Usuarios, cuotas, aptos y accesos

```mermaid
classDiagram
direction LR

class User {
  - id: UUID
  - firstName: String
  - lastName: String
  - documentNumber: String
  - birthDate: Date
  - email: String
  - phone: String
  - passwordHash: String
  - photoUrl: String
  - role: UserRole
  - status: UserStatus
  - hasTemporaryPassword: Boolean
  - createdAt: DateTime
  - updatedAt: DateTime
  + updateContact(email: String, phone: String): void
  + changePassword(passwordHash: String): void
  + assignTemporaryPassword(passwordHash: String): void
  + activate(): void
  + deactivate(): void
  + canAuthenticate(): Boolean
}

class MemberProfile {
  - id: UUID
  - emergencyContactName: String
  - emergencyContactPhone: String
  + updateEmergencyContact(name: String, phone: String): void
}

class TrainerProfile {
  - id: UUID
  - specialty: String
  - description: String
  + updateProfessionalProfile(specialty: String, description: String): void
}

class MembershipPrice {
  - id: UUID
  - amount: Decimal
  - effectiveFrom: DateTime
  - createdAt: DateTime
  + isEffectiveAt(date: DateTime): Boolean
}

class Payment {
  - id: UUID
  - amount: Decimal
  - method: String
  - receiptNumber: String
  - status: PaymentStatus
  - createdAt: DateTime
  - accreditedAt: DateTime
  - expiresAt: DateTime
  - voidedAt: DateTime
  - voidReason: String
  + accredit(accreditedAt: DateTime): void
  + voidPayment(reason: String, voidedAt: DateTime): void
  + calculateExpiration(): DateTime
  + isValidAt(date: DateTime): Boolean
}

class MedicalCertificate {
  - id: UUID
  - fileUrl: String
  - status: MedicalCertificateStatus
  - uploadedAt: DateTime
  - reviewedAt: DateTime
  - reviewComment: String
  + approve(reviewedAt: DateTime): void
  + reject(comment: String, reviewedAt: DateTime): void
  + isApproved(): Boolean
}

class Branch {
  - id: UUID
  - name: String
  - description: String
  - imageUrl: String
  - address: String
  - openingHours: String
  - phone: String
  - latitude: Decimal
  - longitude: Decimal
  - isActive: Boolean
  + updateInformation(): void
  + activate(): void
  + deactivate(): void
}

class AccessPoint {
  - id: UUID
  - name: String
  - qrToken: String
  - isActive: Boolean
  + matchesQrToken(token: String): Boolean
  + activate(): void
  + deactivate(): void
}

class AccessLog {
  - id: UUID
  - roleAtAttempt: UserRole
  - result: AccessResult
  - denialReason: AccessDenialReason
  - attemptedAt: DateTime
}

class UserAuditLog {
  - id: UUID
  - action: UserAuditAction
  - reason: String
  - occurredAt: DateTime
}

class MembershipService {
  + calculateStatus(member: MemberProfile, at: DateTime): MembershipStatus
  + calculateDaysRemaining(member: MemberProfile, at: DateTime): Integer
  + calculateGracePeriodEnd(member: MemberProfile): DateTime
}

class AccessService {
  + validateAccess(user: User, accessPoint: AccessPoint, attemptedAt: DateTime): AccessLog
}

User "1" *-- "0..1" MemberProfile : owns
User "1" *-- "0..1" TrainerProfile : owns
MemberProfile "1" -- "0..*" Payment : has
MemberProfile "1" -- "0..*" MedicalCertificate : submits
User "1" -- "0..*" AccessLog : attempts
Branch "1" *-- "0..*" AccessPoint : contains
Branch "0..1" -- "0..*" AccessLog : receives
AccessPoint "0..1" -- "0..*" AccessLog : records
User "1" -- "0..*" UserAuditLog : auditedUser
User "1" -- "0..*" UserAuditLog : performedBy
User "1" -- "0..*" MembershipPrice : createdBy
User "1" -- "0..*" Payment : createdBy
User "0..1" -- "0..*" Payment : confirmedBy
User "0..1" -- "0..*" Payment : voidedBy
User "0..1" -- "0..*" MedicalCertificate : reviewedBy
MembershipService ..> MemberProfile : uses
MembershipService ..> Payment : uses
AccessService ..> User : uses
AccessService ..> AccessPoint : uses
AccessService ..> MembershipService : uses
AccessService ..> MedicalCertificate : uses
AccessService ..> AccessLog : creates
```

## Sedes, cronogramas y entrenadores

```mermaid
classDiagram
direction LR

class User {
  - id: UUID
  - status: UserStatus
}

class TrainerProfile {
  - id: UUID
  - specialty: String
  - description: String
  + updateProfessionalProfile(specialty: String, description: String): void
}

class Branch {
  - id: UUID
  - name: String
  - isActive: Boolean
  + activate(): void
  + deactivate(): void
}

class WeeklySchedule {
  - id: UUID
  - weekStartsOn: Date
  - createdAt: DateTime
  + copyToWeek(weekStartsOn: Date): WeeklySchedule
}

class ScheduledClass {
  - id: UUID
  - activity: String
  - startsAt: DateTime
  + reschedule(startsAt: DateTime): void
  + assignTrainer(trainer: TrainerProfile): void
  + removeTrainer(): void
  + changeBranch(branch: Branch): void
}

class ScheduleService {
  + createClass(schedule: WeeklySchedule, branch: Branch, trainer: TrainerProfile): ScheduledClass
  + copyPreviousWeek(previous: WeeklySchedule, weekStartsOn: Date): WeeklySchedule
}

User "1" *-- "0..1" TrainerProfile : owns
TrainerProfile "0..*" -- "0..*" Branch : worksAt
WeeklySchedule "1" *-- "0..*" ScheduledClass : contains
WeeklySchedule "0..1" -- "0..*" WeeklySchedule : copiedFrom
Branch "1" -- "0..*" ScheduledClass : hosts
TrainerProfile "0..1" -- "0..*" ScheduledClass : teaches
ScheduleService ..> WeeklySchedule : uses
ScheduleService ..> ScheduledClass : creates
ScheduleService ..> Branch : validates
ScheduleService ..> TrainerProfile : validates
```

## Eventos, novedades y notificaciones

```mermaid
classDiagram
direction LR

class User {
  - id: UUID
  - role: UserRole
}

class Event {
  - id: UUID
  - title: String
  - description: String
  - startsAt: DateTime
  - location: String
  - imageUrl: String
  - status: EventStatus
  + publish(): void
  + cancel(): void
  + isFinished(at: DateTime): Boolean
}

class NewsPost {
  - id: UUID
  - title: String
  - content: String
  - imageUrl: String
  - audience: PublicationAudience
  - status: PublicationStatus
  - publishedAt: DateTime
  + publish(publishedAt: DateTime): void
  + updateContent(title: String, content: String): void
  + deactivate(): void
}

class Notification {
  - id: UUID
  - title: String
  - message: String
  - type: NotificationType
  - createdAt: DateTime
  - readAt: DateTime
  + markAsRead(readAt: DateTime): void
  + isRead(): Boolean
}

class NotificationService {
  + notifyMembershipPriceChange(price: MembershipPrice): void
  + notifyMedicalReview(certificate: MedicalCertificate): void
  + notifyClassChange(scheduledClass: ScheduledClass): void
  + notifyEventCancellation(event: Event): void
}

User "1" -- "0..*" Event : createdBy
User "1" -- "0..*" NewsPost : createdBy
User "1" -- "0..*" Notification : receives
NotificationService ..> User : selectsRecipients
NotificationService ..> Notification : creates
NotificationService ..> MembershipPrice : uses
NotificationService ..> MedicalCertificate : uses
NotificationService ..> ScheduledClass : uses
NotificationService ..> Event : uses
```

## Enumeraciones del dominio

```mermaid
classDiagram
class UserRole {
  <<enumeration>>
  MEMBER
  TRAINER
  ADMIN
}
class UserStatus {
  <<enumeration>>
  ACTIVE
  INACTIVE
}
class MembershipStatus {
  <<enumeration>>
  CURRENT
  EXPIRING_SOON
  EXPIRED
}
class PaymentStatus {
  <<enumeration>>
  ACCREDITED
  VOIDED
}
class MedicalCertificateStatus {
  <<enumeration>>
  PENDING
  APPROVED
  REJECTED
}
class AccessResult {
  <<enumeration>>
  ALLOWED
  DENIED
}
class AccessDenialReason {
  <<enumeration>>
  INVALID_QR
  INACTIVE_USER
  INACTIVE_BRANCH
  INACTIVE_ACCESS_POINT
  EXPIRED_MEMBERSHIP
  MEDICAL_CERTIFICATE_REQUIRED
}
class UserAuditAction {
  <<enumeration>>
  CREATED
  UPDATED
  ACTIVATED
  DEACTIVATED
  PASSWORD_RESET
}
class EventStatus {
  <<enumeration>>
  DRAFT
  PUBLISHED
  CANCELLED
}
class PublicationAudience {
  <<enumeration>>
  ALL
  MEMBERS
  TRAINERS
}
class PublicationStatus {
  <<enumeration>>
  DRAFT
  PUBLISHED
  INACTIVE
}
class NotificationType {
  <<enumeration>>
  MEMBERSHIP_PRICE_CHANGED
  MEMBERSHIP_EXPIRING
  MEMBERSHIP_EXPIRED
  MEDICAL_CERTIFICATE_REVIEWED
  CLASS_CHANGED
  EVENT_CANCELLED
  GENERAL
}
```

## Decisiones de modelado

- Los atributos son privados y las modificaciones se realizan mediante operaciones públicas relevantes.
- Los identificadores de otras clases no se duplican como atributos: las asociaciones representan esas referencias.
- `MemberProfile`, `TrainerProfile`, `AccessPoint` y `ScheduledClass` se modelan mediante composición porque no tienen sentido de dominio sin su objeto contenedor.
- Pagos, aptos, registros, sedes, entrenadores y contenido mantienen identidad e historial propios; por eso utilizan asociaciones normales.
- `MembershipService`, `AccessService`, `ScheduleService` y `NotificationService` coordinan reglas que involucran varias entidades y no almacenan datos propios.
- `MembershipStatus` se calcula a partir de pagos acreditados; no se guarda como un atributo persistente.
- Un pago genera 30 días de vigencia sin acumular días anteriores. Al anularlo, se recalcula desde los pagos acreditados que continúen válidos.
- El período inicial de 20 días se calcula desde el primer pago acreditado que permanezca válido.
- Un apto aprobado no tiene vencimiento.
- El QR identifica un punto de acceso fijo; el usuario se identifica mediante su sesión.
- Cuando el QR no puede identificarse, el registro de acceso no se asocia con una sede ni con un punto de acceso.
- Las clases son informativas y no modelan reservas, cupos, listas de espera ni asistencia.
- Los eventos utilizan una ubicación libre y no se asocian obligatoriamente con una sede.

## Verificación de la consigna

| Criterio | Cumplimiento |
|---|---|
| Responsabilidad única por clase | Las entidades conservan su propio estado y los servicios coordinan reglas entre entidades. |
| Clases e interfaces en PascalCase | Todos los nombres de clases y enumeraciones usan PascalCase. |
| Métodos en camelCase y con verbo | Todas las operaciones comienzan con un verbo y expresan una acción concreta. |
| Atributos privados en camelCase | Todos los atributos usan `-` y camelCase. |
| Tipos de datos indicados | Cada atributo, parámetro y retorno declara su tipo. |
| Relaciones correctas | Se distinguen composición, asociación y dependencia. |
| Multiplicidades completas | Cada asociación indica multiplicidad en ambos extremos. |
| Entidades no duplicadas | Las clases repetidas entre bloques son referencias visuales al mismo concepto, no entidades diferentes. |

## Decisión pendiente

M-Team debe confirmar los medios de pago aceptados. Hasta entonces, `Payment.method` permanece como `String` y no se define una enumeración que agregue valores no aprobados por el alcance.
