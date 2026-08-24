# Diagrama de clases de M-Team

El modelo se presenta mediante vistas temáticas para mantenerlo legible. Todas las vistas representan el mismo dominio y las clases repetidas son el mismo concepto, mostrado solamente con los miembros relevantes para cada módulo.

Los atributos son privados, las operaciones públicas relevantes comienzan con un verbo y todas las asociaciones indican multiplicidad en ambos extremos. No se muestran claves foráneas porque pertenecen al modelo relacional.

## 1. Usuarios y perfiles

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
  - photoUrl: String [0..1]
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

class UserAuditLog {
  <<immutable>>
  - id: UUID
  - action: UserAuditAction
  - reason: String [0..1]
  - occurredAt: DateTime
}

User "1" *-- "0..1" MemberProfile : owns
User "1" *-- "0..1" TrainerProfile : owns
User "1" -- "0..*" UserAuditLog : auditedUser
User "1" -- "0..*" UserAuditLog : performedBy
note for User "{xor: MemberProfile, TrainerProfile}\nMEMBER requires MemberProfile\nTRAINER requires TrainerProfile\nADMIN requires neither profile"
```

## 2. Cuota, pagos y aptos médicos

```mermaid
classDiagram
direction LR

class User {
  - id: UUID
  - role: UserRole
}

class MemberProfile {
  - id: UUID
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
  - reviewedAt: DateTime [0..1]
  - reviewComment: String [0..1]
  + approve(reviewedAt: DateTime): void
  + reject(comment: String, reviewedAt: DateTime): void
  + isApproved(): Boolean
}

MemberProfile "1" -- "0..*" Payment : has
MemberProfile "1" -- "0..*" MedicalCertificate : submits
User "1" -- "0..*" MembershipPrice : createdBy
User "1" -- "0..*" Payment : createdBy
User "1" -- "0..*" Payment : confirmedBy
User "0..1" -- "0..*" Payment : voidedBy
User "0..1" -- "0..*" MedicalCertificate : reviewedBy
```

El estado de cuota `CURRENT`, `EXPIRING_SOON` o `EXPIRED` se calcula desde los pagos acreditados y no se almacena como atributo.

## 3. Control de acceso

```mermaid
classDiagram
direction LR

class User {
  - id: UUID
  - role: UserRole
  - status: UserStatus
  + canAuthenticate(): Boolean
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
  <<immutable>>
  - id: UUID
  - roleAtAttempt: UserRole
  - result: AccessResult
  - denialReason: AccessDenialReason [0..1]
  - attemptedAt: DateTime
}

User "1" -- "0..*" AccessLog : attempts
Branch "1" *-- "0..*" AccessPoint : contains
Branch "0..1" -- "0..*" AccessLog : receives
AccessPoint "0..1" -- "0..*" AccessLog : records
```

`Branch` y `AccessPoint` son opcionales en un registro cuando el QR es inexistente o fue alterado. La validación completa del ingreso coordina usuario, cuota, apto, sede y punto de acceso; se implementará en la capa de servicios y no como una entidad adicional.

## 4. Sedes, cronogramas y entrenadores

```mermaid
classDiagram
direction LR

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

TrainerProfile "0..*" -- "0..*" Branch : worksAt
WeeklySchedule "1" *-- "0..*" ScheduledClass : contains
WeeklySchedule "0..1" -- "0..*" WeeklySchedule : copiedFrom
Branch "1" -- "0..*" ScheduledClass : hosts
TrainerProfile "0..1" -- "0..*" ScheduledClass : teaches
```

Las clases son informativas: no incluyen reservas, cupos, listas de espera ni control de asistencia.

## 5. Eventos, novedades y notificaciones

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
  - imageUrl: String [0..1]
  - status: EventStatus
  + publish(): void
  + cancel(): void
  + isFinishedAt(date: DateTime): Boolean
}

class NewsPost {
  - id: UUID
  - title: String
  - content: String
  - imageUrl: String [0..1]
  - audience: PublicationAudience
  - status: PublicationStatus
  - publishedAt: DateTime [0..1]
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
  - readAt: DateTime [0..1]
  + markAsRead(readAt: DateTime): void
  + isRead(): Boolean
}

User "1" -- "0..*" Event : createdBy
User "1" -- "0..*" NewsPost : createdBy
User "1" -- "0..*" Notification : receives
```

Los eventos utilizan una ubicación libre. Las notificaciones se generan desde servicios de aplicación ante aumentos de cuota, vencimientos, revisiones de aptos, cambios de clases y cancelaciones de eventos.

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

## Reglas de modelado

- Los atributos son privados y solo se muestran operaciones públicas relevantes; no se incluyen getters ni setters triviales.
- `UserAuditLog` y `AccessLog` no tienen métodos porque son registros históricos inmutables, creados por los servicios y utilizados para consulta.
- Los servicios, controladores, repositorios y DTO se documentarán en el diseño de capas; no son entidades del dominio.
- `MemberProfile`, `TrainerProfile`, `AccessPoint` y `ScheduledClass` utilizan composición porque dependen conceptualmente de su contenedor.
- Pagos, aptos, registros, sedes, entrenadores y publicaciones conservan identidad e historial propios y utilizan asociaciones normales.
- El precio de la cuota es único para todos los socios y conserva su historial.
- Cada pago acreditado genera 30 días de vigencia sin acumular días anteriores.
- Al anular un pago, la vigencia y el período inicial se recalculan utilizando los pagos acreditados que permanezcan válidos.
- El período inicial para ingresar sin apto aprobado dura 20 días desde el primer pago acreditado válido.
- Un apto aprobado permanece válido sin fecha de vencimiento.
- El QR identifica un punto de acceso fijo y el usuario se identifica mediante su sesión.
- Desactivar una entidad no elimina sus relaciones ni registros históricos.

## Decisión pendiente

M-Team debe confirmar los medios de pago aceptados. Hasta entonces, `Payment.method` permanece como `String` y no se agrega una enumeración con valores no aprobados.
