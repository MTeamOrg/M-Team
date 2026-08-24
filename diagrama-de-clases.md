# Diagrama de clases de M-Team

Esta primera versión representa los conceptos del negocio definidos en el alcance. No incluye controladores, servicios, repositorios, JWT ni componentes de la interfaz.

## Núcleo operativo

```mermaid
classDiagram
direction LR

class User {
  +UUID id
  +string firstName
  +string lastName
  +string documentNumber
  +date birthDate
  +string email
  +string phone
  +string passwordHash
  +string photoUrl
  +UserRole role
  +UserStatus status
  +boolean mustChangePassword
  +datetime createdAt
  +datetime updatedAt
}

class MemberProfile {
  +UUID id
  +UUID userId
  +string emergencyContactName
  +string emergencyContactPhone
}

class TrainerProfile {
  +UUID id
  +UUID userId
  +string specialty
  +string description
}

class MembershipPrice {
  +UUID id
  +decimal amount
  +datetime effectiveFrom
  +UUID createdById
  +datetime createdAt
}

class Payment {
  +UUID id
  +UUID memberId
  +decimal amount
  +string method
  +string receiptNumber
  +PaymentStatus status
  +UUID createdById
  +datetime createdAt
  +UUID confirmedById
  +datetime accreditedAt
  +datetime expiresAt
  +UUID voidedById
  +datetime voidedAt
  +string voidReason
}

class MedicalCertificate {
  +UUID id
  +UUID memberId
  +string fileUrl
  +MedicalCertificateStatus status
  +datetime uploadedAt
  +UUID reviewedById
  +datetime reviewedAt
  +string reviewComment
}

class Branch {
  +UUID id
  +string name
  +string description
  +string imageUrl
  +string address
  +string openingHours
  +string phone
  +decimal latitude [optional]
  +decimal longitude [optional]
  +boolean isActive
}

class AccessPoint {
  +UUID id
  +UUID branchId
  +string name
  +string qrToken
  +boolean isActive
}

class AccessLog {
  +UUID id
  +UUID userId
  +UserRole roleAtAttempt
  +UUID branchId [optional]
  +UUID accessPointId [optional]
  +AccessResult result
  +AccessDenialReason denialReason
  +datetime attemptedAt
}

class WeeklySchedule {
  +UUID id
  +date weekStartsOn
  +UUID copiedFromId
  +datetime createdAt
}

class ScheduledClass {
  +UUID id
  +UUID scheduleId
  +UUID branchId
  +UUID trainerId
  +string activity
  +datetime startsAt
}

class TrainerBranch {
  +UUID trainerId
  +UUID branchId
}

class UserAuditLog {
  +UUID id
  +UUID userId
  +UUID performedById
  +UserAuditAction action
  +string reason
  +datetime occurredAt
}

User "1" *-- "0..1" MemberProfile : has
User "1" *-- "0..1" TrainerProfile : has
MemberProfile "1" --> "0..*" Payment : owns
MemberProfile "1" --> "0..*" MedicalCertificate : uploads
User "1" --> "0..*" AccessLog : attempts
Branch "1" *-- "0..*" AccessPoint : contains
Branch "0..1" --> "0..*" AccessLog : receives
AccessPoint "0..1" --> "0..*" AccessLog : records
WeeklySchedule "1" *-- "0..*" ScheduledClass : contains
WeeklySchedule "0..1" --> "0..*" WeeklySchedule : copied into
Branch "1" --> "0..*" ScheduledClass : hosts
TrainerProfile "0..1" --> "0..*" ScheduledClass : teaches
TrainerProfile "1" --> "0..*" TrainerBranch : works at
Branch "1" --> "0..*" TrainerBranch : has trainers
User "1" --> "0..*" UserAuditLog : is audited
User "1" --> "0..*" UserAuditLog : performs
User "1" --> "0..*" MembershipPrice : creates
User "1" --> "0..*" Payment : creates confirms or voids
User "1" --> "0..*" MedicalCertificate : reviews
```

## Comunicación y contenido

```mermaid
classDiagram
direction LR

class User {
  +UUID id
  +UserRole role
}

class Event {
  +UUID id
  +string title
  +string description
  +datetime startsAt
  +string location
  +string imageUrl
  +EventStatus status
  +UUID createdById
}

class NewsPost {
  +UUID id
  +string title
  +string content
  +string imageUrl
  +PublicationAudience audience
  +PublicationStatus status
  +datetime publishedAt
  +UUID createdById
}

class Notification {
  +UUID id
  +UUID userId
  +string title
  +string message
  +NotificationType type
  +datetime createdAt
  +datetime readAt
}

User "1" --> "0..*" Event : manages
User "1" --> "0..*" NewsPost : publishes
User "1" --> "0..*" Notification : receives
```

## Estados del dominio

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

## Reglas que no deben modelarse como datos duplicados

- El estado de la cuota se calcula en el backend a partir del último pago acreditado: `CURRENT`, `EXPIRING_SOON` o `EXPIRED`.
- `expiresAt` se fija en 30 días desde `accreditedAt`; un pago nuevo reemplaza la vigencia anterior y no acumula días.
- Si se anula un pago, la vigencia se recalcula utilizando el pago acreditado más reciente que continúe válido.
- Si el pago anulado era el primer pago válido, el período inicial de 20 días se recalcula desde la acreditación válida más antigua que permanezca registrada.
- Durante los primeros 20 días desde el primer pago acreditado, un socio puede ingresar sin apto aprobado.
- Un apto aprobado permanece vigente sin fecha de vencimiento.
- Un entrenador no necesita cuota ni apto médico para ingresar, pero su cuenta debe estar activa.
- Desactivar entidades conserva sus relaciones e historial.
- Los estados `UPCOMING` y `FINISHED` de un evento se derivan de la fecha; `CANCELLED` sí se persiste.
- El usuario se identifica mediante su sesión y escanea el QR fijo de un `AccessPoint`; no existe un QR personal por usuario.
- Si el QR es inexistente o fue alterado, el intento se registra con motivo `INVALID_QR` y sin sede ni punto de acceso asociados.
- Las clases son informativas: no incluyen reservas, cupos, listas de espera ni control de asistencia.

## Decisiones pendientes de validación

1. Confirmar los medios de pago aceptados por M-Team.

Por cumplimiento estricto del alcance, los eventos utilizan una ubicación libre y no se vinculan con una sede. Cada nueva carga de un apto rechazado se conserva en el historial, como establece APM-07. Los administradores utilizan los datos comunes de `User`, ya que el alcance no define atributos adicionales para un perfil administrativo.
