# Frontend Spec: CPA (ผู้สอบบัญชี) Integration — Draft v0.9

อิงจาก: 

- Draft (UI vision): [draft.md](draft.md)
- Backend spec: [../../../gms-backend/spec/cpa-joined/spec.md](../../../gms-backend/spec/cpa-joined/spec.md) (**Rev 3.6 — phase 1–8 implemented**)
- Reference docx (ในโฟลเดอร์นี้):
  - [รายงานผลการตรวจสอบ-ผู้สอบบัญชีโครงการ.docx](รายงานผลการตรวจสอบ-ผู้สอบบัญชีโครงการ.docx) — CPA Checklist
  - [แบบประเมินผู้สอบบัญชีโครงการ.docx](แบบประเมินผู้สอบบัญชีโครงการ.docx) — Evaluation form

> **Convention** (เดียวกับ backend): `cpa` ใช้ใน code/identifier เท่านั้น — UI text ใช้ **"ผู้สอบบัญชี"** เสมอ.
>
> - **CPA Master** → "ผู้สอบบัญชี · รับผิดชอบ"
> - **CPA Reviewer** → "ผู้สอบบัญชี · ผู้ช่วย"
> - กลุ่ม section / menu → "ผู้สอบบัญชี"

> **สถานะ**: v0.9 — Phase 1+2+3 implemented. มี frontend feature flag (env `VITE_CPA_JOINED_ENABLED`) คู่ขนานกับ backend. Backend change request ค้าง: รับ `date` query param ใน `POST /cpas/import` (ดู §3.1.1)

---

## 0. Change Log

### v0.9 (ปัจจุบัน) — **Reintroduce frontend feature flag** + Phase 3 implemented

ความเข้าใจเดิม (v0.2+): frontend ไม่ต้องมี flag — backend gate ทุกอย่าง. **ผิด**:

- Backend `milestone.cpaRequired` ถูก gate ด้วย `cpa_joined_enabled` ฝั่ง backend → ตอน flag OFF จะคืน `false` เสมอ (ไม่ใช่ "ตามกฎ business rule")
- ถ้า frontend เชื่อตรงๆ → flag OFF ตลอด rollout = ทุกงวดจะถูกมองว่า "ไม่ต้องตรวจ CPA" ซึ่งผิด — งวดที่ตามกฎ budget ต้องเป็น CPA-type จะถูก treat แบบ skip flow
- ฝั่ง frontend ยังต้องสามารถบอกได้ว่า "งวดนี้ตามกฎเดิมต้องตรวจ CPA" ก่อน flag จะเปิด

**Resolution**: เพิ่ม frontend flag คู่ขนานกับ backend:

- **Env var**: `VITE_CPA_JOINED_ENABLED` (default `"false"` ทุก environment)
- **Hook**: [`useCpaJoinedEnabled()`](src/libs/hooks/useCpaJoinedEnabled.ts) คืน boolean
- **`useCpaContext`** เพิ่ม `cpaJoinedEnabled` ใน context และ:
  - flag ON → `cpaRequired = milestone.cpaRequired` (backend authoritative)
  - flag OFF → `cpaRequired = needCcOrCpa(milestone, milestones, financeData) === "CPA"` (legacy frontend logic)
  - flag OFF → gate ทุก `can*` permission เป็น false → UI ใหม่ทั้งหมดถูกซ่อน
- **UI components** gate โดย flag:
  - `<CpaSection>` early-return null ถ้า flag OFF
  - Partner project list badge "รอตอบรับคำเชิญผู้สอบบัญชี" ใน `Info.tsx` — hide ถ้า flag OFF
  - Phase 4 components (`<MilestoneCpaInviteBox>`, `<CpaForwardButton>`, CPA tabs) — gate เช่นกัน

**Workflow rollout**:

1. Merge frontend code พร้อม `VITE_CPA_JOINED_ENABLED="false"` — เห็นเหมือนเดิม
2. Backend เปิด `cpa_joined_enabled=true` ก่อน (per environment)
3. Frontend เปิด `VITE_CPA_JOINED_ENABLED="true"` — UI ใหม่ขึ้น

### v0.8 — Phase 2 implemented + import polish (date + template) + backend change request

- Phase 2 (Registry) implemented: `cpa.ts/cpa.type.ts` service, `CpaList.tsx`, route mount, sidebar menu — tsc + lint pass
- เพิ่ม **date input ใน import modal** (match Expert UI pattern) + ปุ่ม "ตัวอย่าง (.csv)" download
- สร้างไฟล์ template ที่ [public/media/cpa_import_template.csv](public/media/cpa_import_template.csv) — header ไทยตาม `_FIELD_ALIASES` ของ backend
- **Backend change request** ⚠️ — `POST /cpas/import` ต้องรับ query param `date=YYYY-MM-DD` แล้วใช้เป็น `recorded_at` แทน `today` (frontend ส่ง `?date=...` ไว้แล้ว)

### v0.7 — Phase 1 implemented + tighten phase dependency

- Phase 1 (Foundation) implemented: enums, permission, type extensions, `useCpaContext` hook — tsc + lint pass
- §14 Phases — เพิ่ม **hard dependency note** + คอลัมน์ `depends on` ในตาราง phase
- ระบุชัดเจน: **Phase 2 → 3** เพราะ `<CpaInviteModal>` ต้องใช้ `useQueryCpasPickable` จาก `cpa.ts` ของ Phase 2 + ต้องมี CPA จริงใน DB ผ่าน import flow ก่อน end-to-end ของ invite จะใช้ได้

### v0.6 — CpaChecklist / CpaEvaluation เป็น hash section ใน Finance.tsx (ไม่ใช่ route, ไม่ใช่ sub-tab nav แยก)

ตรวจ [Finance.tsx](src/components/milestones/MilestoneDetail/Finance.tsx) แล้วพบว่ามี **tab + hash navigation อยู่ภายในตัวเอง** อยู่แล้ว (สมุดรายวันฯ / สรุปงบ / รายงานการเงิน / **Compliance Checklist** / รายงานสรุปปิดโครงการ). Tab `Compliance Checklist` มี `hidden: needCcOrCpaType !== "CC"`.

- **ไม่สร้าง route ใหม่** — Cpa\* ไม่ใช่ sub-route ของ Finance
- ไม่ต้อง `Finance(parentRoute).addChildren([...])` ที่ออกแบบไว้ใน v0.5 — ทิ้งทั้งหมด
- **เพิ่ม 2 tab ใน `tabItems` ของ Finance.tsx** ที่ระดับเดียวกับ `section--compliance-checklist`:
  - `section--cpa-checklist` — "รายงานผลการตรวจสอบ" — `hidden: !milestone.cpaRequired`
  - `section--cpa-evaluation` — "แบบประเมินผู้สอบบัญชี" — `hidden: !milestone.cpaRequired`
- **Section content** render ใน Finance.tsx ด้วย pattern เดิม (`display: section === "cpa-checklist" ? "" : "none"`) คู่กับ ComplianceChecklist ที่มีอยู่
- **CC vs CPA mutually exclusive**: `needCcOrCpaType` คืน "CC" หรือ "CPA" — tab จะแสดงเฉพาะที่ตรง context (CC tab ซ่อนเมื่อ CPA, CPA tabs ซ่อนเมื่อ CC) — ไม่ขึ้นพร้อมกัน
- ผลกระทบกับ §10.2 partner badge → hash anchor ใช้ `#cpa-section` (อยู่บน ParticipantList — คนละหน้า, ไม่ชน hash ของ Finance.tsx)

### v0.5 — CpaChecklist / CpaEvaluation ย้ายมาเป็น sub-tab ใต้ Finance

- Route: child ของ `Finance(route)` ไม่ใช่ sibling ของ Finance อีกต่อไป (path เปลี่ยนเป็น `/.../finance/cpa-checklist`, `/.../finance/cpa-evaluation`)
- Tab menu: ไม่เพิ่มใน [AppNav.tsx](src/components/milestones/MilestoneDetail/AppNav.tsx) (ที่ระดับ Content/Finance) — เพิ่มเป็น sub-tab ใน [Finance.tsx](src/components/milestones/MilestoneDetail/Finance.tsx) แทน
- เหตุผล: CPA Checklist + Evaluation เป็นเรื่องของลำดับงานทางการเงิน (อยู่ใน Finance workflow lane) — จัดให้ navigation สื่อ semantic ตรงกว่า ลดจำนวน top-level tab ด้วย

### v0.4 — embed class names ตรง code + ปิด open items ทั้งหมด

- ปรับ shape ของ `project.cpaInvitation` → ใช้ชื่อ class ตาม code: **`CpaInvitationProjectEmbed`** (top) + **`CpaInvitationCpaEmbed`** (nested) — เดิม spec เขียนเป็น `CpaInvitationSummary` ไม่ตรง implementation
- เพิ่ม `cpa.userId: number | null` ใน nested embed (backend §3.10.2 ระบุชัดแล้ว)
- ลบ email-fallback logic — `useCpaContext` match ด้วย `cpaInvitation.cpa.userId === auth.user.id` เท่านั้น (clean)
- ปิด open items ทั้ง 2 ข้อใน §13 (เลิกมี open items)
- `className="cpa-checklist"` marker — **คงไว้** (user confirm), ไม่ต้องลบหลัง refactor

### v0.3 — sync กับ backend Rev 3.6

Backend phase 8 ส่งของที่ frontend ต้องใช้ครบ — spec รอบนี้ปรับให้อิง data จริงที่ backend ส่งมา:

| เรื่อง                            | v0.2 (เดิม assumption)                                         | v0.3 (confirmed Rev 3.6)                                                                                                                                                                                                |
| --------------------------------- | -------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `milestone.cpaRequired`           | ขอ backend ส่งมา                                               | ✅ **ส่งมาแล้ว** ([backend §3.10.1](../../../gms-backend/spec/cpa-joined/spec.md))                                                                                                                                      |
| `project.cpaInvitation`           | embed ที่ทั้ง milestone + project                              | ✅ **embed เฉพาะ project-level** (Rev 3.5 binding; active row เดียว INVITED **หรือ** ACCEPTED) — ลบของ milestone-level ออก                                                                                              |
| Badge "รอตอบรับ" บนรายการโครงการ  | ต้อง query `useQueryMyCpaInvitations({ status: INVITED })` แยก | ✅ **ไม่ต้อง query แยก** — CPA Master เป็น participant ตั้งแต่ INVITED → เห็นโครงการในรายการ partner เลย, อ่าน `project.cpaInvitation.status === INVITED` มา derive badge                                               |
| CPA ก่อน accept เป็น participant? | ก่อนตอบไม่เป็น (ต้อง pending list แยก)                         | ✅ **เป็น participant ตั้งแต่ INVITED** — listener `sync_cpa_master_participant` ตอนนี้ subscribe `CpaInvitedEvent` ด้วย                                                                                                |
| ACL guard `status === ACCEPTED`   | -                                                              | ⚠️ **เพิ่มเข้ามาใหม่** — write actions (เขียน checklist, approve `finance_cpa`, จัดการทีม Reviewer) ต้องการ ACCEPTED. Frontend ต้อง mirror gate นี้ใน UI (ปุ่ม hide/disable). Master INVITED แค่ "read + accept/reject" |
| CPA registry batch by date        | unsure                                                         | ✅ **ส่งมาแล้ว** — `GET /cpa-batches` + `GET /cpas?date=YYYY-MM-DD` (pattern Expert)                                                                                                                                    |
| Cancel/reject/withdraw note       | optional                                                       | ✅ **บังคับฝั่ง UI** (backend รับ optional แต่ frontend require) — Mantine form `isNotEmpty` + submit disabled                                                                                                          |
| Reviewer self-edit                | ?                                                              | ✅ ห้าม edit `email` + `identityId` (ผูกกับ EEF Connect), แต่ **name / phone แก้ได้**                                                                                                                                   |
| Badge click navigation            | deep-link หรือ open page                                       | ✅ **deep-link ด้วย hash** (`#cpa-section`) ถ้าไม่ชน id อื่น; ถ้าชน fallback เปิด page ปกติ                                                                                                                             |
| (8) Section component             | ต้อง trace ในโค้ด                                              | ✅ user mark ด้วย `className="cpa-checklist"` ใน [FinanceReport.tsx](src/components/milestones/MilestoneDetail/FinanceReport.tsx) — frontend ต้อง refactor ออกมาเป็น component ใหม่ (`<MilestoneCpaInviteBox>`)         |

### v0.2 — sync กับ draft.md ใหม่ (ลบ feature flag, ใช้ Mantine table แทน RJSF, mount section ใต้ ParticipantList, etc.)

### v0.1 — ฉบับแรก (RJSF / inbox route แยก / feature flag ฝั่ง frontend)

---

## 1. Backend Contract — Confirmed Shapes (Rev 3.6)

### 1.1 Milestone payload — [milestone.type.ts](src/libs/services/milestone.type.ts)

```ts
type Milestone = {
  // ...เดิม

  // ✅ backend ส่งมาแล้ว — compute จาก needs_cpa_step() + flag gate
  cpaRequired: boolean

  // ✅ workflow status 6 fields (null ถ้า cpaRequired = false)
  financeCpaStatus?: ApproveStatus | null
  financeCpaStatusAt?: string | null
  financeCpaStatusByRoles?: string[] | null
  financeCpaForwardStatus?: ApproveStatus | null
  financeCpaForwardStatusAt?: string | null
  financeCpaForwardStatusByRoles?: string[] | null

  // ❌ ไม่มี milestone.cpaInvitation — binding เป็น project-level (อ่านจาก project.cpaInvitation)
}
```

### 1.2 Project payload — extend [proposal.type.ts](src/libs/services/proposal.type.ts) (หรือ project type)

ชื่อ class ตาม backend code ([app/cpa/domain/repo.py](../../../gms-backend/app/cpa/domain/repo.py)) — **ใช้ชื่อเดียวกันใน frontend เพื่อให้ trace ข้าม repo ง่าย**:

```ts
type CpaInvitationCpaEmbed = {
  id: number
  licenseNo: string
  title?: string | null
  firstName: string
  lastName: string
  email: string
  phone?: string | null
  userId?: number | null // ⭐ frontend match ด้วย field นี้ (null ถ้า CPA ยังไม่ login SSO ครั้งแรก)
}

type CpaInvitationProjectEmbed = {
  id: number
  status: "INVITED" | "ACCEPTED" // active เท่านั้น — ไม่มี REJECTED/WITHDRAWN/SUPERSEDED
  milestoneNo: number // ที่ issued (ใช้ audit; binding ครอบทุกงวด)
  cpa: CpaInvitationCpaEmbed
  invitedAt: string
  respondedAt?: string | null
}

type Project = {
  // ...เดิม
  cpaInvitation?: CpaInvitationProjectEmbed | null // null = ไม่มี active binding
}
```

> Rev 3.5 binding: 1 active per project (partial unique index บน DB). History invitation อ่านผ่าน `/cpa-invitations?project_id=X` เมื่อจำเป็น (ไม่ embed)

### 1.3 User payload — **ไม่ต้องขยาย**

CPA login → participant อยู่แล้วตั้งแต่ INVITED → โครงการขึ้นใน "my projects" ปกติ → frontend อ่าน `project.cpaInvitation.status` เพื่อ derive badge "รอตอบรับ". ไม่ต้อง add `user.cpaId` หรือ `user.pendingCpaInvitations`.

> Match identity: `auth.user.id === project.cpaInvitation.cpa.userId` (`userId` มาจาก nested embed §1.2). กรณี `cpa.userId === null` = CPA ยังไม่เคย login SSO → ก็แปลว่าไม่ใช่ user ปัจจุบันที่ login อยู่ — เงื่อนไข match จะ false โดยอัตโนมัติ ไม่ต้องเขียน guard เพิ่ม

### 1.4 Approval steps — ไม่ต้องแก้

backend ส่ง `title`, `extra_status`, `approvers`, `possible_you`, `role` มาให้พร้อม render ตามนั้น

---

## 2. Constants / Enums

### 2.1 [src/libs/constants/role.ts](src/libs/constants/role.ts)

```ts
export enum Role {
  // ...เดิม
  CPA = "cpa",
}

export enum ParticipantRole {
  // ...เดิม
  CPA_MASTER   = "project.cpa.master",
  CPA_REVIEWER = "project.cpa.reviewer",
}

[Role.CPA]: "ผู้สอบบัญชี",
[ParticipantRole.CPA_MASTER]:   "ผู้สอบบัญชี · รับผิดชอบ",
[ParticipantRole.CPA_REVIEWER]: "ผู้สอบบัญชี · ผู้ช่วย",
```

### 2.2 [src/libs/constants/permission.ts](src/libs/constants/permission.ts)

```ts
export enum CpaPermission {
  REGISTRY_MANAGE = "cpa.registry.manage",
}
```

### 2.3 Invitation status enum

```ts
// src/libs/services/cpaInvitation.type.ts
export enum CpaInvitationStatus {
  INVITED = "INVITED",
  ACCEPTED = "ACCEPTED",
  REJECTED = "REJECTED",
  WITHDRAWN = "WITHDRAWN",
  SUPERSEDED = "SUPERSEDED",
}
```

---

## 3. Service Layer

### 3.1 `cpa.ts` + `cpa.type.ts` — Registry (admin)

```ts
GetQueryCpas(search: GetCpaListParams)       // GET /cpas?date=...
GetQueryCpaBatches()                         // GET /cpa-batches — date dropdown
GetQueryCpaId(id)

useQueryCpas(search)
useQueryCpaBatches()
useQueryCpa(id)
useCreateCpaMutation()                       // (Phase 1: UI อาจ disable — ใช้ import)
useUpdateCpaMutation()                       // (Phase 1: UI อาจ disable)
useDeleteCpaMutation()                       // soft delete
useImportCpasMutation()                      // { file: FormData, date: "YYYY-MM-DD" } → POST /cpas/import?date=<date>
useDownloadCpaReportLevel1Mutation()
useDownloadCpaReportLevel2Mutation()
useDownloadCpaReportLevel3Mutation()

// picker (autocomplete) — ใช้ใน invite modal
GetQueryCpasPickable(search)                 // GET /cpas?search=&status=active (ไม่ส่ง date)

invalidateCpas()
invalidateCpaBatches()
```

Types: `Cpa`, `CpaListItem`, `GetCpaListParams { date?, keyword?, status? }`, `CreateCpaPayload`, `UpdateCpaPayload`, `CpaStatus`, `CpaBatch { date, total }`, `ImportCpasResponse { success, created, updated, errors }`.

#### 3.1.1 Backend change request — `POST /cpas/import` ต้องรับ `date`

⚠️ **Pending backend update** (v0.8): ปัจจุบัน backend hardcode `recorded_at = today` ([import_cpas.py](../../../gms-backend/app/cpa/api/import_cpas.py)). Frontend ใช้ pattern เดียวกับ Expert — ให้ admin **เลือกวันที่** เป็น snapshot date (เช่นเลือกย้อนหลังตอนนำเข้าข้อมูลที่ได้รับมาก่อนแต่กรอกทีหลัง).

Required backend change:

```python
# app/cpa/api/import_cpas.py
def _import_cpas(
    file: UploadFile,
    date: Date | None = None,                     # ← เพิ่ม query param
    uac: UserAccessControl = Depends(...),
):
    ...
    result = import_cpas_usecase(uow, uac, ImportCpasIn(rows=parsed, recorded_at=date))
```

```python
# app/cpa/usecase/import_cpas.py
@dataclass(frozen=True)
class ImportCpasIn:
    rows: list[ImportCpaRow]
    recorded_at: date | None = None              # ← เพิ่ม

# ใน loop: Cpa.create(..., recorded_at=input.recorded_at or date.today())
# ใน existing: existing.update_profile(..., recorded_at=input.recorded_at)
```

Frontend ส่งเป็น query param `?date=YYYY-MM-DD` แล้วใน [cpa.ts:useImportCpasMutation](src/libs/services/cpa.ts). จนกว่า backend จะ patch — date param ที่ส่งมาจะถูก ignore + ใช้ today.

#### 3.1.2 Template CSV

ไฟล์: [public/media/cpa_import_template.csv](public/media/cpa_import_template.csv) — header ไทยตรงกับ `_FIELD_ALIASES` ใน backend `parse_row` ([import_cpas.py](../../../gms-backend/app/cpa/usecase/import_cpas.py)):

```
เลขทะเบียน, คำนำหน้า, ชื่อ, นามสกุล, ที่อยู่, อีเมล, เบอร์โทรศัพท์,
จำนวนชั่วโมง, ผลการทดสอบ, วันที่เริ่มขึ้นทะเบียน, เลขบัตรประชาชน
```

ปุ่ม "ตัวอย่าง (.csv)" ใน CpaList → download ตรงจาก `/public/media/...` (pattern ExpertList).

### 3.2 `cpaInvitation.ts` + `cpaInvitation.type.ts` — Lifecycle

```ts
// queries
GetQueryCpaInvitations(search) // admin/officer
GetQueryProjectCpaInvitations(projectId) // history ของโครงการ (ผ่าน /cpa-invitations?project_id=X)

useQueryCpaInvitations(search)
useQueryProjectCpaInvitations(projectId)

// mutations — ทุกตัวรับ `note: string` (frontend require non-empty สำหรับ cancel/reject/withdraw)
useInviteCpaMutation() // POST /projects/{id}/milestones/{no}/cpa-invitation { cpa_id, note? }
useCancelCpaInvitationMutation() // DELETE /cpa-invitations/{invId} { note: required }
useAcceptCpaInvitationMutation() // POST /cpa-invitations/{invId}/accept (ไม่ต้องการ note)
useRejectCpaInvitationMutation() // POST /cpa-invitations/{invId}/reject { note: required }
useWithdrawCpaInvitationMutation() // POST /cpa-invitations/{invId}/withdraw { note: required }
```

> **ไม่มี `useQueryMyCpaInvitations`** — เลิกใช้ pending-list query แล้ว (Rev 3.6 ใส่ participant ตั้งแต่ INVITED, CPA เห็นใน project list ปกติ)

Types: `CpaInvitation`, `CpaInvitationListResponse`.

### 3.3 `cpaEvaluation.ts` + `cpaEvaluation.type.ts` — Clinic ประเมิน

```ts
GetQueryCpaEvaluation(proposalId, milestoneNo)
useQueryCpaEvaluation(proposalId, milestoneNo)
useCreateCpaEvaluationMutation()
useUpdateCpaEvaluationMutation()
```

Types: `CpaEvaluation { q1Score, q2Score, q3Score, q4Score, totalScore, remark, evaluatedAt, evaluatedBy }`.

### 3.4 ขยาย service เดิม

| File                                                     | Change                                                                     |
| -------------------------------------------------------- | -------------------------------------------------------------------------- | ----- |
| [milestone.type.ts](src/libs/services/milestone.type.ts) | + `cpaRequired`, `financeCpa*` (3 fields), `financeCpaForward*` (3 fields) |
| [proposal.type.ts](src/libs/services/proposal.type.ts)   | + `Project.cpaInvitation: CpaInvitationSummary                             | null` |
| [proposal.ts](src/libs/services/proposal.ts)             | **ไม่ต้องเพิ่ม endpoint** — reuse participant CRUD สำหรับ Reviewer         |
| [auth.type.ts](src/libs/services/auth.type.ts)           | **ไม่ต้องแก้** (Rev 3.6 ไม่ขยาย user payload)                              |

---

## 4. Routes

### 4.1 Admin (system-management)

ใน [src/routeTree/d.tsx](src/routeTree/d.tsx) — pattern เดียวกับ `expertIndexRoute`:

```ts
const cpaIndexRoute = createRoute({
  getParentRoute: () => d,
  path: "d/$departmentId/system-management/cpas",
}) as AnyRoute
cpaIndexRoute.addChildren([CpaList(cpaIndexRoute)])

const cpaReportsRoute = createRoute({
  getParentRoute: () => d,
  path: "d/$departmentId/system-management/cpas/reports",
}) as AnyRoute
cpaReportsRoute.addChildren([CpaReports(cpaReportsRoute)])
```

### 4.2 Milestone routes — **ไม่มี route ใหม่สำหรับ Cpa\***

CpaChecklist + CpaEvaluation เป็น **hash section ใน Finance.tsx** (pattern เดียวกับ `#section--compliance-checklist` ที่มีอยู่) — ไม่ใช่ route แยก:

- ไม่แก้ [src/components/milestones/MilestoneDetail/index.tsx](src/components/milestones/MilestoneDetail/index.tsx) (`route.addChildren([Content, Finance])` คงเดิม)
- ไม่ใช้ `createRoute` สำหรับ CpaChecklist/CpaEvaluation
- Hash URL: `#section--cpa-checklist`, `#section--cpa-evaluation` — เปลี่ยนผ่าน `history.pushState` เหมือน tab อื่นๆ ของ Finance.tsx
- ดู §6.3 สำหรับการเพิ่ม tab + render section

---

## 5. `useCpaContext()` Hook — Single Source of Truth สำหรับ permission/visibility

`src/libs/hooks/useCpaContext.ts`:

```ts
type CpaContextOpts = {
  project?: Project // ใช้ project.cpaInvitation + project.participants
  milestone?: Milestone // ใช้ milestone.cpaRequired + finance_cpa* status
}

function useCpaContext({ project, milestone }: CpaContextOpts) {
  const { auth } = useAuth()
  const user = auth.user

  const invitation = project?.cpaInvitation // ⚠️ active row only (INVITED|ACCEPTED)

  // identity of current user wrt invitation — match ด้วย cpa.userId (nested embed §1.2)
  // ถ้า invitation.cpa.userId === null (CPA ยังไม่เคย login SSO) → expression = false โดยธรรมชาติ
  const isMaster = !!invitation && invitation.cpa.userId === user?.id

  const isMasterAccepted = isMaster && invitation?.status === "ACCEPTED"
  const isMasterInvited = isMaster && invitation?.status === "INVITED"
  const isMasterAccepted = isMaster && invitation?.status === "ACCEPTED"
  const isMasterInvited = isMaster && invitation?.status === "INVITED"

  // Reviewer = participant role=CPA_REVIEWER + user_id match
  const isReviewer =
    project?.participants.some(
      (p) => p.role === ParticipantRole.CPA_REVIEWER && p.userId === user?.id,
    ) ?? false

  // admin/global
  const isAdmin = user?.permissions.includes(AdminPermission.WAND) ?? false
  const canManageRegistry =
    user?.permissions.includes(CpaPermission.REGISTRY_MANAGE) ?? false

  // partner identity
  const isPartner =
    !isMaster &&
    !isReviewer &&
    (project?.participantIds?.includes(user?.id ?? -1) ?? false)

  return {
    // identity
    isMaster,
    isMasterAccepted,
    isMasterInvited,
    isReviewer,
    isAdmin,
    isPartner,
    canManageRegistry,

    // milestone gate
    cpaRequired: milestone?.cpaRequired ?? false,

    // write/read permissions (mirror backend ACL §3.8 Rev 3.6)
    canWriteCpaChecklist: isMasterAccepted || isReviewer || isAdmin, // ⚠️ ต้อง ACCEPTED ของ Master ก่อน Reviewer เขียนได้ (เพราะ Reviewer existence depend on Master ACCEPTED)
    canApproveCpaStep: isMasterAccepted || isAdmin, // Reviewer ห้าม approve
    canManageReviewers: isMasterAccepted || isAdmin, // Master INVITED ห้ามเพิ่ม Reviewer
    canRespondInvitation: isMasterInvited || isAdmin, // accept/reject
    canWithdraw: isMasterAccepted || isAdmin,
    canInviteCpa: (isPartner || isAdmin) && !invitation, // ภาคีเชิญได้เมื่อไม่มี active binding
    canCancelInvitation:
      (isPartner || isAdmin) && invitation?.status === "INVITED",
    canSubmitToClinic:
      (isPartner || isAdmin) &&
      milestone?.financeCpaStatus === "APPROVED" &&
      milestone?.financeCpaForwardStatus === "PENDING",
  }
}
```

> **กฎสำคัญ** (Rev 3.6 ACL guard §3.8): Master ระหว่าง INVITED **ดูได้แต่เขียนไม่ได้** — ห้ามเขียน checklist, ห้าม approve, ห้ามเพิ่ม Reviewer. Backend จะ reject ถ้าหลุดผ่านมา; frontend ใช้ hook นี้ hide/disable UI ก่อน

---

## 6. Components

```
src/components/cpa/
├── registry/
│   ├── CpaList.tsx              # pattern ExpertList (filter date dropdown, import, table)
│   ├── CpaImportModal.tsx       # dropzone + date
│   ├── CpaCreateEditModal.tsx   # Phase 1: ปุ่มอาจซ่อนไว้ก่อน
│   └── CpaReports.tsx           # 3 ปุ่ม download
├── invitation/
│   ├── CpaSection.tsx           # section "รายการผู้สอบบัญชี" ใน ParticipantList
│   ├── CpaInviteModal.tsx       # picker + invite
│   ├── CpaInvitationActions.tsx # ปุ่ม accept/reject/withdraw/cancel
│   ├── ConfirmActionWithNoteModal.tsx  # confirm + textarea note (required)
│   ├── CpaStatusBadge.tsx
│   └── CpaTeamRow.tsx
├── milestone/
│   ├── MilestoneCpaInviteBox.tsx       # ใช้ใน FinanceReport "(8)" + ใช้ซ้ำใน CpaSection (เชิญ)
│   ├── CpaChecklist.tsx                # tab — ลอก ComplianceChecklist
│   ├── CpaChecklistData.ts             # static map (sections + items + deps)
│   ├── CpaChecklistRow.vm.ts           # MobX-Keystone row vm
│   ├── CpaChecklist.vm.ts              # parent vm (rows + opinion + dates)
│   ├── CpaEvaluation.tsx               # tab — Mantine form 4 ข้อ
│   ├── CpaEvaluation.vm.ts
│   └── CpaForwardButton.tsx            # ปุ่ม "ส่งให้คลินิก"
└── shared/
    ├── CpaPicker.tsx                   # autocomplete
    └── useCpaContext.ts                # §5
```

### 6.1 หน้า admin: `CpaList.tsx`

- Layout เหมือน [ExpertList.tsx](src/components/expert/ExpertList.tsx)
- Header "ผู้สอบบัญชี" + ปุ่ม "นำเข้า"
- Dropdown "ข้อมูล ณ วันที่" (ใช้ `useQueryCpaBatches()`) — default batch ล่าสุด
- ตาราง: เลขทะเบียน, ชื่อ-นามสกุล, อีเมล, โทร, ที่อยู่, ชั่วโมงอบรม, คะแนนเฉลี่ย (admin เท่านั้น), สถานะ
- Phase 1: ปุ่มเพิ่ม/ลบ/แก้รายเดี่ยว — **hidden ไว้ก่อน** (draft §10 — เน้น import flow)
- Gate: `<Guard condition={u => u.permissions.includes(CpaPermission.REGISTRY_MANAGE)}>`

### 6.2 Section "รายการผู้สอบบัญชี" ใน ParticipantList

Mount ใน [src/components/participant/ParticipantList.tsx](src/components/participant/ParticipantList.tsx) ใต้รายการเจ้าหน้าที่:

```
[Title: รายการเจ้าหน้าที่โครงการ]
  ภาคี A (ผู้รับผิดชอบโครงการ)
  ...

[Title: รายการผู้สอบบัญชี]   ← <CpaSection />   (anchor id="cpa-section")
  ผู้สอบบัญชี C (ผู้สอบบัญชี · รับผิดชอบ)        [Badge: รอตอบรับ / ทำงาน / ถอนตัวแล้ว]
                                              [actions ตาม persona + state]
    └── ผู้สอบบัญชี D (ผู้สอบบัญชี · ผู้ช่วย)
    └── ผู้สอบบัญชี E (ผู้สอบบัญชี · ผู้ช่วย)
  [+ ปุ่ม เชิญผู้สอบบัญชี — ภาคี/admin, เมื่อไม่มี active binding]
  [+ ปุ่ม เพิ่มผู้ช่วย — Master/admin, **เมื่อ ACCEPTED เท่านั้น**]
```

#### 6.2.1 Visibility

- ภาคี/admin/officer: เห็นเมื่อโครงการมี ≥1 งวด `cpaRequired = true` (อ่านจาก project.milestones)
- CPA Master/Reviewer: เห็นเสมอ (เพราะเข้าถึงโครงการได้ก็เพราะเป็นทีม CPA)
- Persona อื่น: hide section

#### 6.2.2 Action matrix — แยกตาม invitation status

| Action                            | ภาคี (canInviteCpa/canCancelInvitation) | Master INVITED (canRespondInvitation) | Master ACCEPTED                                | Reviewer                               | Admin |
| --------------------------------- | --------------------------------------- | ------------------------------------- | ---------------------------------------------- | -------------------------------------- | ----- |
| เชิญผู้สอบบัญชี                   | ✅ ถ้าไม่มี active binding              | -                                     | -                                              | -                                      | ✅    |
| ยกเลิกคำเชิญ (ก่อน accept)        | ✅ ถ้า status=INVITED                   | -                                     | -                                              | -                                      | ✅    |
| ตอบรับคำเชิญ                      | -                                       | ✅                                    | -                                              | -                                      | ✅    |
| ปฏิเสธคำเชิญ                      | -                                       | ✅                                    | -                                              | -                                      | ✅    |
| ถอนตัว                            | -                                       | -                                     | ✅                                             | -                                      | ✅    |
| เพิ่ม Reviewer                    | -                                       | ❌ (ห้าม Rev 3.6 ACL)                 | ✅                                             | -                                      | ✅    |
| แก้ไข Reviewer (name/phone)       | -                                       | -                                     | ✅ (เฉพาะทีมตัวเอง)                            | ✅ (เฉพาะ self, ห้าม email+identityId) | ✅    |
| แก้ไข Reviewer (email/identityId) | -                                       | -                                     | ✅ (เฉพาะทีมตัวเอง — ก่อน confirm participant) | ❌                                     | ✅    |
| ลบ Reviewer                       | -                                       | -                                     | ✅ (เฉพาะทีมตัวเอง)                            | ❌ (ปลดตัวเองไม่ได้)                   | ✅    |
| ลบ Master                         | -                                       | -                                     | -                                              | -                                      | ✅    |

> **Reviewer self-edit** (Rev 3.6 user answer): name + phone แก้ตัวเองได้; email + identityId แก้ไม่ได้ (ผูกกับ EEF Connect). ใช้ `useCpaContext` + check `participant.user_id === user.id && role === CPA_REVIEWER` → ใน `<UpdateParticipantModal>` ส่ง prop `lockedFields=["email", "identityId"]`

#### 6.2.3 Status badge mapping

```ts
const statusBadge = {
  INVITED: { color: "yellow", label: "รอตอบรับ" },
  ACCEPTED: { color: "green", label: "ทำงาน" },
  // backend ส่งเฉพาะ active (INVITED|ACCEPTED) ใน project.cpaInvitation
  // history (REJECTED/WITHDRAWN/SUPERSEDED) อ่านจาก /cpa-invitations?project_id=X ถ้าจะแสดง
}
```

### 6.3 Hash section + tab ใน Finance.tsx — pattern เดียวกับ `section--compliance-checklist`

**ไม่แตะ [AppNav.tsx](src/components/milestones/MilestoneDetail/AppNav.tsx)** (ที่ระดับบนยังเป็น "ผลการดำเนินงาน" / "การเงิน"). เพิ่มเฉพาะใน [Finance.tsx](src/components/milestones/MilestoneDetail/Finance.tsx).

#### 6.3.1 เพิ่ม 2 รายการใน `tabItems`

ที่ [Finance.tsx:522-574](src/components/milestones/MilestoneDetail/Finance.tsx) — เพิ่มถัดจาก `section--compliance-checklist` (line 553-563):

```tsx
// ...tabItems เดิม
{
  id: "section--compliance-checklist",
  title: "Compliance Checklist",
  active: section === "compliance-checklist",
  onClick: async () => { ... },
  hidden: needCcOrCpaType !== "CC",
},

// ← เพิ่ม 2 รายการนี้
{
  id: "section--cpa-checklist",
  title: "รายงานผลการตรวจสอบ",
  active: section === "cpa-checklist",
  onClick: async () => {
    await invalidateQueries()
    history.pushState(null, "", `#section--cpa-checklist`)
    resetScrollInModal()
  },
  hidden: !milestone.cpaRequired,
},
{
  id: "section--cpa-evaluation",
  title: "แบบประเมินผู้สอบบัญชี",
  active: section === "cpa-evaluation",
  onClick: async () => {
    await invalidateQueries()
    history.pushState(null, "", `#section--cpa-evaluation`)
    resetScrollInModal()
  },
  hidden: !milestone.cpaRequired,
},

// section--final-report เดิมต่อท้าย
```

> **CC vs CPA mutually exclusive**:
>
> - `needCcOrCpaType === "CC"` → "Compliance Checklist" tab แสดง, CPA 2 tab ซ่อน
> - `milestone.cpaRequired === true` (= `needCcOrCpaType === "CPA"`) → CC tab ซ่อน, CPA 2 tab แสดง
> - ไม่มีกรณีที่ขึ้นพร้อมกัน เพราะ backend `needs_cpa_step()` กับ frontend `needCcOrCpa()` mutually exclusive

#### 6.3.2 Render section content

ที่ [Finance.tsx:1086](src/components/milestones/MilestoneDetail/Finance.tsx#L1086) ที่มี `display: section === "compliance-checklist" ? "" : "none"` — เพิ่ม section ใหม่อีก 2 บล็อกตาม pattern เดียวกัน:

```tsx
<Box style={{ display: section === "compliance-checklist" ? "" : "none" }}>
  <ComplianceChecklist ... />
</Box>

{/* ← เพิ่ม */}
<Box style={{ display: section === "cpa-checklist" ? "" : "none" }}>
  <CpaChecklist
    journalFormVm={journalFormVm}   /* หรือ vm ใหม่ที่ branch ออกมา */
    milestone={milestone}
    project={project}
    onChange={...}
    readOnly={...}
  />
</Box>

<Box style={{ display: section === "cpa-evaluation" ? "" : "none" }}>
  <CpaEvaluation
    proposalId={proposalId}
    milestoneNo={milestoneNo}
    project={project}
  />
</Box>
```

> **mount strategy**: ใช้ `display: none` (ไม่ใช่ conditional unmount) ตามที่ ComplianceChecklist ทำ — รักษา state ของ vm + scroll position เมื่อสลับ tab. **ยกเว้น** ถ้า CpaEvaluation มี side effect ตอน mount (เช่น auto-create draft) อาจ lazy-mount ด้วย `{section === "cpa-evaluation" && <CpaEvaluation ... />}` ก็ได้

#### 6.3.3 Section parsing (hash)

[Finance.tsx](src/components/milestones/MilestoneDetail/Finance.tsx) มี logic `const section = ...` อยู่แล้ว (parse hash `#section--XXX` → `XXX`). ไม่ต้องแก้ตรงนี้ — แค่เพิ่ม case `cpa-checklist` / `cpa-evaluation` รองรับโดย parser เดิม

### 6.4 `<MilestoneCpaInviteBox>` — refactor "(8)" section

User mark element ด้วย `className="cpa-checklist"` ใน [FinanceReport.tsx](src/components/milestones/MilestoneDetail/FinanceReport.tsx) → frontend ต้อง:

1. แยก element นี้ออกมาเป็น component ใหม่ `<MilestoneCpaInviteBox proposalId milestone project />`
2. Logic ภายใน:

| `milestone.cpaRequired` | `project.cpaInvitation` | UI                                                                                                                                                                |
| ----------------------- | ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| false                   | -                       | ไม่แสดง (เดิมเป็น slot แนบเอกสาร CPA → ลบไปเลย เพราะกฎ CPA workflow มาแทน)                                                                                        |
| true                    | null                    | Panel "ต้องเชิญผู้สอบบัญชีมาตรวจรายงาน" + ปุ่ม "เชิญผู้สอบบัญชี" (เปิด `<CpaInviteModal>`)                                                                        |
| true                    | status=INVITED          | Panel "รอผู้สอบบัญชีตอบรับ" + ชื่อผู้รับเชิญ + ปุ่ม "ยกเลิกคำเชิญ" (ภาคี/admin เท่านั้น)                                                                          |
| true                    | status=ACCEPTED         | Panel "ผู้สอบบัญชี: [ชื่อ]" + link "ดูรายงานผลการตรวจสอบ" → hash `#section--cpa-checklist` (เปลี่ยน tab ใน Finance.tsx) + แสดง `financeCpaStatus` (Mantine Badge) |

> Component นี้ใช้ใน FinanceReport "(8)" จุดเดียว. **ปุ่ม "เชิญผู้สอบบัญชี" ใน CpaSection ของ ParticipantList ใช้ logic เดียวกันแต่ render context ต่าง** — สกัด helper `useCpaInviteAction(project, milestone?)` ให้ shared logic อยู่ที่เดียว, ส่วน rendering แยกตามที่

### 6.5 `<CpaForwardButton>` — ปุ่ม "ส่งให้คลินิก"

- แสดงใน [Finance.tsx](src/components/milestones/MilestoneDetail/Finance.tsx) เหนือ FinanceApproveStepper เป็น callout
- เงื่อนไข render: `useCpaContext().canSubmitToClinic`
- กด → `<ConfirmActionWithNoteModal>` (ใช้ confirm pattern เดียวกัน แต่ note optional — ส่งต่อไม่ใช่ negative)
- Submit → mutation approve step `finance_cpa_forward` (ผ่าน `useApproveReportMutation` ที่มีอยู่)

### 6.6 FinanceApproveStepper

**ไม่แก้** — backend ส่ง step + title + status มาให้แล้ว

---

## 7. UI Flow ตาม Persona

### 7.1 ภาคี (Partner)

1. ทำรายงานก้าวหน้า → กรอกสมุดรายวัน → "รายงานการเงิน" → "(8) แนบเอกสารสำคัญฯ"
2. ถ้า `milestone.cpaRequired = true` → `<MilestoneCpaInviteBox>` แสดง
3. กด "เชิญผู้สอบบัญชี" (จาก "(8)" หรือจาก section "รายการผู้สอบบัญชี" ก็ได้) → `<CpaInviteModal>` → submit
4. รอ Master ตอบรับ:
   - ระหว่างรอ → กด submit รายงานการเงิน → backend reject → toast (Rev 3 gate)
   - คลิกผิด → ยกเลิกคำเชิญ → modal + textarea note (**required**) → submit
5. หลัง ACCEPTED → submit รายงานการเงินได้ → workflow ไป `finance_cpa`
6. Master approve `finance_cpa` → `<CpaForwardButton>` ปรากฏ → กด → modal confirm → ส่งคลินิก

### 7.2 ผู้สอบบัญชี · รับผิดชอบ (CPA Master)

1. Login ผ่าน EEF Connect → backend link `cpa.user_id` (SSO hook §6 backend)
2. **เข้าหน้ารายการโครงการของภาคี** (เพราะ Rev 3.6 listener สร้าง participant ตั้งแต่ INVITED) → เห็นโครงการที่ถูกเชิญในรายการเลย
3. โครงการที่มี `cpaInvitation.status === INVITED` AND `cpaInvitation.cpa` คือตัวเอง → **แสดง badge "รอตอบรับ"** บน row ของรายการโครงการ (สีเหลือง, ตำแหน่งใกล้ชื่อโครงการ)
4. กด row → navigate ไปหน้าโครงการ + hash `#cpa-section` → auto scroll ไปที่ section "รายการผู้สอบบัญชี" ใน ParticipantList
5. ที่ row ของตัวเอง (Master) → ปุ่ม **ตอบรับคำเชิญ** + **ปฏิเสธ**
   - ตอบรับ → confirm modal (note ไม่บังคับ) → POST accept → badge เปลี่ยนเป็น "ทำงาน"
   - ปฏิเสธ → modal + textarea note (**required**) → POST reject → ถูกถอดจาก participants (cascade) → หายจากรายการโครงการ
6. หลัง ACCEPTED → ปุ่ม "เพิ่มผู้ช่วย" ปรากฏ → กด → modal (reuse `<UpdateParticipantModal>` ที่ lock role=CPA_REVIEWER) → identity_id + email + ชื่อ
7. เข้า milestone → tab **"รายงานผลการตรวจสอบ"** → กรอกตาราง
8. Finance tab → approve `finance_cpa` (ผ่าน `ApproveReportMenu` เดิม)
9. ถอนตัว → ที่ row ตัวเองใน CpaSection → ปุ่ม "ถอนตัว" → modal + note (**required**) → POST withdraw → ระบบถอด Master + Reviewers ทุกคน

### 7.3 ผู้สอบบัญชี · ผู้ช่วย (CPA Reviewer)

1. รับ email "คุณถูกเพิ่มเป็นผู้สอบบัญชี · ผู้ช่วย" → confirm participant (flow เดิม)
2. Login → เห็นโครงการ
3. CpaSection → row ตัวเอง: ไม่มีปุ่ม "ลบ" หรือ "ถอนตัว" (เฉพาะ Master ลบ Reviewer ได้)
4. แก้ตัวเอง (ผ่าน `<UpdateParticipantModal>`) → name/phone แก้ได้, email/identityId **disabled + tooltip** "ใช้เชื่อมกับ EEF Connect ไม่สามารถแก้ไขได้"
5. tab "รายงานผลการตรวจสอบ" → กรอก checklist ได้ (ต้องการ `isMasterAccepted` ของ project = true)
6. ไม่เห็นปุ่ม approve/revise `finance_cpa`

### 7.4 คลินิกการเงิน (COST_CENTER_SUPPORTER)

1. หลังภาคีกดส่ง → milestone เข้าคิว clinic
2. tab "แบบประเมินผู้สอบบัญชี" → กรอก 4 ข้อ + remark
3. กลับ tab การเงิน → กด approve `finance_review`
   - ถ้ายังไม่ประเมิน: ปุ่ม disabled + tooltip (UI guard เสริม backend reject)

### 7.5 Admin (wand)

1. Sidebar "จัดการระบบ" → "ผู้สอบบัญชี" → CpaList + filter date + import
2. หน้า /cpas/reports → 3 ปุ่ม download
3. ในโครงการ — ทำได้ทุก action ตาม §6.2.2 (override permission ทั้งหมด)

---

## 8. CPA Checklist (form) — ลอก ComplianceChecklist pattern

### 8.1 ไฟล์ที่ลอก

| Reference                                                                                                                                  | New                                                  |
| ------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------- |
| [ComplianceChecklist.tsx](src/components/milestones/MilestoneDetail/ComplianceChecklist.tsx)                                               | `src/components/cpa/milestone/CpaChecklist.tsx`      |
| [ComplianceChecklistData.tsx](src/components/milestones/MilestoneDetail/ComplianceChecklistData.tsx)                                       | `src/components/cpa/milestone/CpaChecklistData.ts`   |
| [ComplianceChecklistRow.vm.ts](src/components/forms/CostCenter/ReportFrom/ComplianceChecklistRow.vm.ts)                                    | `src/components/cpa/milestone/CpaChecklistRow.vm.ts` |
| (parent vm — `JournalForm.vm.complianceChecklistMap` ใน [JournalForm.vm.ts](src/components/forms/CostCenter/ReportFrom/JournalForm.vm.ts)) | `src/components/cpa/milestone/CpaChecklist.vm.ts`    |

### 8.2 โครงสร้าง section (จาก backend §8 + docx)

```
หัวข้อ 1: การเงินและหลักฐานการจ่ายเงิน           22 items
หัวข้อ 2: การบันทึกรายการบัญชี และเอกสารหลักฐาน   7 items
หัวข้อ 3: การหักภาษี ณ ที่จ่าย                   3 items
หัวข้อ 4: การตรวจสอบครุภัณฑ์                     5 items
หัวข้อ 5: การนำเสนอรายการ ...                     7 items
```

Schema เดียว M = L (backend Rev 3.1). Item text รายบรรทัด → extract จาก [docx](รายงานผลการตรวจสอบ-ผู้สอบบัญชีโครงการ.docx) ตอน implement Phase 6.

### 8.3 Header fields (Phase 6.2)

- `auditedAt`, `documentsReceivedAt`, `documentsAdditionalReceivedAt` — Buddhist-era date picker
- `opinion` — textarea

> draft §31: "เริ่มที่ตารางเลย" → **Phase 6.1 ทำตาราง, Phase 6.2 header**

### 8.4 Storage + Sync

- เก็บใน `milestone.report.finance_data.cpaChecklist` (backend §2.5)
- save ผ่าน `PATCH /projects/{id}/milestones/{no}/reports` (endpoint เดิม)
- ถ้า JournalForm vm bind YJS อยู่แล้ว → CpaChecklist vm ก็ sync ตามใน scope finance_data เดียวกัน

### 8.5 Permission

- Write: `canWriteCpaChecklist` (Master ACCEPTED + Reviewer + Admin) — Rev 3.6 ACL
- Read: ทุกคนที่เข้าโครงการ
- Lock เมื่อ `finance_cpa_status = APPROVED`

### 8.6 Approve / Revise

- ปุ่มอยู่ที่ Finance tab (ไม่ใช่ Checklist tab) ใน `ApproveReportMenu`
- Revise ทำได้แม้ checklist ยังไม่เสร็จ (draft §32)
- Approve เช็ค `vm.isValid` ฝั่ง frontend ก่อน mutation

---

## 9. CPA Evaluation (form) — Mantine ธรรมดา

### 9.1 โครงสร้าง

จาก [แบบประเมินผู้สอบบัญชีโครงการ.docx](แบบประเมินผู้สอบบัญชีโครงการ.docx) + backend §2.3:

- 4 ข้อ q1..q4 — Radio group 1-5
- `totalScore = (q1+q2+q3+q4) * 5` — read-only
- `remark` — textarea (optional)

### 9.2 Permission

- Write: COST_CENTER_SUPPORTER + admin
- Read: ทุกคนที่เข้าโครงการ
- Lock เมื่อ `finance_review_status = APPROVED`

---

## 10. Navigation/Menu Changes

### 10.1 Officer sidebar [OfficerMenu.tsx](src/components/layouts/OfficerMenu.tsx)

ใต้ "จัดการระบบ" หลังจาก "ผู้ทรงคุณวุฒิ":

```ts
{
  label: "ผู้สอบบัญชี",
  href: `/d/${departmentId}/system-management/cpas`,
  permissions: [CpaPermission.REGISTRY_MANAGE],
},
```

### 10.2 Partner project list — badge "รอตอบรับ" (Rev 3.6 ใหม่)

> **ไม่ต้อง query แยก** — backend ใส่ participant ตั้งแต่ INVITED → โครงการขึ้นใน "my projects" → frontend อ่าน `project.cpaInvitation` มา derive badge

ใน list component ของ partner home (ต้อง grep หา exact file — เป็น candidate: [src/components/partners/](src/components/partners/)):

```tsx
{
  project.cpaInvitation?.status === "INVITED" &&
    project.cpaInvitation.cpa.userId === user?.id && (
      <Badge color="yellow" size="sm">
        รอตอบรับ
      </Badge>
    )
}
```

คลิก row → navigate ไป `/d/.../proposals/{proposalId}/proposal-section#cpa-section` → page scroll ไปที่ section อัตโนมัติ (อิง `<CpaSection id="cpa-section">`)

> ถ้า `cpa-section` id ชนกับ element อื่น → fallback ใช้ id เฉพาะกว่า เช่น `cpa-list-section`

---

## 11. Negative Action — Confirmation Pattern (note required)

Component: `<ConfirmActionWithNoteModal>`

```tsx
type Props = {
  opened: boolean
  onClose: () => void
  title: string // เช่น "ยืนยันการถอนตัว"
  description: string // เช่น "หลังถอนตัว ทีมผู้สอบบัญชี · ผู้ช่วยทั้งหมดจะถูกถอดออกอัตโนมัติ"
  confirmLabel: string // "ถอนตัว" (สี red)
  noteRequired?: boolean // default = true สำหรับ negative actions
  noteLabel?: string // "เหตุผล" / "หมายเหตุ"
  onConfirm: (note: string) => Promise<void>
}
```

Validation:

- `noteRequired = true` (default) → Mantine `useForm` + `validate.note = isNotEmpty("กรุณากรอกเหตุผล")` → ปุ่ม submit disabled จนกว่ามี text
- `noteRequired = false` → ปุ่ม submit enabled เสมอ; textarea optional

Usage:

- Cancel invitation (partner) — `noteRequired=true`, "เหตุผลที่ยกเลิก"
- Reject invitation (CPA Master) — `noteRequired=true`, "เหตุผลที่ปฏิเสธ"
- Withdraw (CPA Master) — `noteRequired=true`, "เหตุผลที่ถอนตัว"
- Delete Reviewer (CPA Master) — `noteRequired=false` (ไม่ใช่ negative-public action)
- Submit to clinic (forward) — `noteRequired=false` (ไม่ใช่ negative)

> **Backend** รับ note เป็น optional — frontend บังคับฝั่งเดียว (Rev 3.6 user answer)

---

## 12. Edge Cases

| Case                                                        | UI behavior                                                                                               |
| ----------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| ภาคี submit finance lane โดยไม่มี ACCEPTED                  | toast error + `<MilestoneCpaInviteBox>` highlight                                                         |
| ภาคีเชิญขณะมี INVITED ค้าง                                  | modal validation: alert + disable submit                                                                  |
| ภาคีเชิญขณะมี ACCEPTED                                      | ปุ่มเชิญ disabled + tooltip "ต้องให้ผู้สอบบัญชีปัจจุบันถอนตัวก่อน"                                        |
| Clinic approve `finance_review` ก่อนประเมิน                 | ปุ่ม disabled + tooltip + (fallback) toast จาก backend                                                    |
| CPA Master ยัง INVITED — พยายามเขียน checklist              | tab "รายงานผลการตรวจสอบ" → ตาราง render แต่ inputs disabled + banner "ต้องตอบรับคำเชิญก่อน" (Rev 3.6 ACL) |
| CPA Master ยัง INVITED — พยายามเพิ่ม Reviewer               | ปุ่ม "เพิ่มผู้ช่วย" hidden                                                                                |
| Reviewer พยายาม approve/revise step                         | ปุ่ม **hidden**                                                                                           |
| CPA Master ถอนตัวขณะ Reviewer กำลังกรอก checklist (อีก tab) | next query refresh → ACL change → tab lock อ่านอย่างเดียว + toast                                         |
| งวดถัดไป (binding) — ภาคีเชิญใหม่                           | ปุ่ม disabled + tooltip "ผู้สอบบัญชีของโครงการนี้คือ [ชื่อ] (binding ระดับโครงการ)"                       |
| `cpaRequired = false`                                       | section "รายการผู้สอบบัญชี" hide + tab CPA hide + กล่อง approval ไม่มี                                    |
| Reviewer แก้ email/identityId ของตัวเอง                     | field disabled + tooltip "ใช้เชื่อมกับ EEF Connect"                                                       |
| `project.cpaInvitation = null` แต่ user เคยเป็น Reviewer    | (case: Master เพิ่ง withdraw → Reviewer ถูก cascade) → query refresh → ไม่เห็นโครงการต่อ                  |

---

## 13. Open Items

**ไม่มี open item ค้าง** — รอบ v0.4 ปิดทั้ง 2 จุดของ v0.3 แล้ว:

1. ~~`cpa.userId` ใน invitation embed~~ → ✅ **มีอยู่แล้ว** ใน `CpaInvitationCpaEmbed.userId` (backend §3.10.2). Frontend match ด้วย `cpaInvitation.cpa.userId === auth.user.id` — ไม่ต้อง fallback email
2. ~~`className="cpa-checklist"` หลัง refactor~~ → ✅ **คงไว้** (user confirm) — frontend dev refactor เป็น `<MilestoneCpaInviteBox>` แต่ไม่ลบ class

---

## 14. Phases (mapped backend)

ทุก phase merge ได้ทันที — backend ปิด flag อยู่บน main, frontend แค่ render ตาม data.

> **Hard dependencies ฝั่ง frontend** (ห้ามข้าม):
>
> - 1 → 2: type + permission + hook ต้องมาก่อน admin UI
> - **2 → 3**: `<CpaInviteModal>` ใช้ `useQueryCpasPickable` จาก `cpa.ts` (Phase 2) + ต้องมี CPA จริงใน DB ผ่าน import (Phase 2 admin) ก่อนภาคีจะ invite ได้
> - 3 → 4: workflow integration อ่าน `project.cpaInvitation` ที่ปกติ mutate ผ่าน invite/cancel/accept (Phase 3)
> - 3 → 5: Reviewer management depends on Master being able to accept invitation
> - 4 → 6: tab `cpa-checklist` แสดงเมื่อ `milestone.cpaRequired` + workflow status visible
> - 7 ขนานได้ — reports + polish ทำตอนไหนก็ได้หลัง phase 2 (admin) + 6 (data ครบ)
>
> **ห้ามข้าม Phase 2 ไป Phase 3** — แม้จะ stub picker query ได้ แต่ end-to-end ของ invite flow ทำไม่ได้ถ้าไม่มี CPA ใน DB

| Backend phase            | Frontend deliverables                                                                                                                                                                                                                                                                  |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1 Foundation             | enums (Role.CPA, ParticipantRole.CPA_MASTER/REVIEWER), CpaPermission, ขยาย Milestone + Project type, `useCpaContext()` hook                                                                                                                                                            |
| 2 Registry               | `cpa.ts` service (incl. `useQueryCpasPickable` — **required by Phase 3**) + `CpaList` + `CpaImportModal` (date + dropzone) + route + sidebar menu                                                                                                                                      |
| 3 Invitation             | `cpaInvitation.ts` service + `CpaSection` + `CpaInviteModal` + `CpaInvitationActions` + `ConfirmActionWithNoteModal` + partner project list badge (อ่านจาก `project.cpaInvitation.status`)                                                                                             |
| 4 Workflow integration   | refactor `<MilestoneCpaInviteBox>` ออกจาก FinanceReport "(8)", `CpaForwardButton` ใน Finance.tsx, FinanceApproveStepper (อาจ cosmetic เท่านั้น)                                                                                                                                        |
| 5 CPA Team (Reviewer)    | extend `UpdateParticipantModal` รองรับ role CPA_REVIEWER + lock email/identityId เมื่อ Reviewer self-edit, `CpaSection` แสดง nested team, guard "เพิ่มผู้ช่วย" ด้วย `isMasterAccepted`                                                                                                 |
| 6 Checklist + Evaluation | `CpaChecklist.tsx` + vm (ลอก Compliance) + tab; `CpaEvaluation.tsx` + tab; guard ปุ่ม approve finance_review (มี evaluation มั้ย). ลำดับย่อย: 6.1 ตาราง checklist, 6.2 header (audit dates + opinion), 6.3 evaluation. ในตาราง checklist Master ที่ INVITED → render disabled + banner |
| 7 Reports + Polish       | `CpaReports.tsx` 3 ปุ่ม, empty states, loading skeletons, copy/UX สุดท้าย                                                                                                                                                                                                              |

---

## 15. References

- **Backend Rev 3.6**: [../../../gms-backend/spec/cpa-joined/spec.md](../../../gms-backend/spec/cpa-joined/spec.md)
- **Mantine table-driven form**: [ComplianceChecklist.tsx](src/components/milestones/MilestoneDetail/ComplianceChecklist.tsx), [ComplianceChecklistRow.vm.ts](src/components/forms/CostCenter/ReportFrom/ComplianceChecklistRow.vm.ts), [ComplianceChecklistData.tsx](src/components/milestones/MilestoneDetail/ComplianceChecklistData.tsx)
- **Admin list + import + batch dropdown**: [ExpertList.tsx](src/components/expert/ExpertList.tsx), [expert.ts](src/libs/services/expert.ts)
- **Participant CRUD**: [ParticipantList.tsx](src/components/participant/ParticipantList.tsx), [UpdateParticipantModal.tsx](src/components/participant/UpdateParticipantModal.tsx), [proposal.ts](src/libs/services/proposal.ts)
- **Milestone tabs + approve menu**: [AppNav.tsx](src/components/milestones/MilestoneDetail/AppNav.tsx), [index.tsx](src/components/milestones/MilestoneDetail/index.tsx)
- **Approval stepper**: [FinanceApproveStepper.tsx](src/components/milestones/MilestoneDetail/FinanceApproveStepper.tsx) (ไม่แก้)
- **Section "(8)" target**: [FinanceReport.tsx](src/components/milestones/MilestoneDetail/FinanceReport.tsx) — มี `className="cpa-checklist"` marker
- **Sidebar menu**: [OfficerMenu.tsx](src/components/layouts/OfficerMenu.tsx)
- **Admin route mount**: [src/routeTree/d.tsx](src/routeTree/d.tsx)
- **Auth/role**: [AuthProvider.tsx](src/components/auth/AuthProvider.tsx), [role.ts](src/libs/constants/role.ts), [permission.ts](src/libs/constants/permission.ts)
