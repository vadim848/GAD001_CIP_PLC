# CIP for external clients: scope and complexity estimate

Basis: the code analysis in `CIP_StateMachine.md` (sections 2–7) and the theory "ReMeat03 CIP v0.1".
Status today: FB_CIP washes only its own loop (`eLocal`), see section 6a there.

## 1. Verdict

**Medium-high complexity.** The building blocks are there: client valve routes in `FB_CIP_Client`, parameter array per client, the dashboard enum, Anybus decoding. The core sequence in FB_CIP, however, was written for the local loop only and has to be extended, not just re-parametrized.

Rough size, assuming the hardware is ready and the open questions in section 4 are answered:

| Part | Effort (PLC work) |
|---|---|
| Code changes (sections 2.1–2.7) | **≈ 4–6 weeks** |
| Commissioning per client route on the plant | **+ 1–2 weeks** (depends on the number of clients and on site access) |

The main uncertainty is not the code but the process and hardware answers in section 4. One wrong assumption about the return path or the sensors can turn a stage into a redesign.

## 2. Work packages

### 2.1 Requests and queue: small (1–2 days)
- `m_TakePosInCIPBuffer`: accept client requests, not only `eLocal`; give a message instead of silently dropping a rejected request.
- HMI: choose client and program (today only two Local buttons).
- Refuse a client that is in production or already queued (interlock with the production side).

### 2.2 CIP state per client: medium (2–3 days)
- Today `eCIP_State` is a single variable inside FB_CIP. Needed: an array per `E_CIP_Usage` (persistent), plus expiry (`eExpired` already exists in `E_Element_State_Long`).
- Production must read it ("may this tank or line be used?"). The former per-element objects (`fbElement[...]`, still referenced in `vis_Main`/`vis_CIP`) are gone from the code; the folder `22 InterConnection` is empty.

### 2.3 Routing to the client: small-medium (2–4 days)
- `fbCIP_Usage.m_Run(eLocal)` is hard-coded. In the Detergent/DisInfection connect steps `fbCIP_Usage` is not called at all, because circulation runs tank → tank. Every connect step needs `m_Run(eReq.eClient)` and a check of the client status.
- The client valves are already wired in MAIN (V1023–V1030, V1106) and `FB_CIP_Client` has the routes Marinade_Up/Low, Color_Up/Low, Drain, Loop, Future. The valve combinations of each route must be verified on the P&ID.

### 2.4 New SFC stages: **large (2–3 weeks), the core of the work**
These are the theory stages that have no SFC steps yet:
- **CIP1 Drain Product Rest** (pipe and sprinkler; stages 1–2): today a stub.
- **Client Fill** (stages 5, 14): push solution into the client line by `r_Volume_Solution_Fill`, with the return to drain until solution arrives (phase separation).
- **Client Condition pipe + sprinkler** (stages 6, 15): flow/temp/concentration/time; split between pipe and sprinkler (`r_Flow_Sprinkler`, `t_Condition*_Sprinkler`, `r_Volume_Sprinkler_Ratio`).
- **Collect / Drain / Clean for the client** (stages 8–11, 17–20): today implemented only for Local, with fixed RPM instead of flow control.

The main SFC has 57 steps today; it will roughly double. The SFC is graphical and can only be edited in the TwinCAT IDE. For maintainability it is worth considering a parametric stage instead of copy-pasting each stage per client.

### 2.5 Parameters and process control: medium (3–4 days)
- `g_Data.arr_CIP_Param[...]` already exists per client, but FB_CIP reads `[E_CIP_Usage.eLocal]` in **58 places**. Replace with `eReq.eClient`.
- Hard-coded values to turn into parameters: T94 < 250, T93 450/500, dilution 900 L, concentration filter 30 s.
- Unused parameters: `r_CIP_Client_Pump`, `r_Volume_Recycle_*`, `r_Volume_Drain_R_Water`.

### 2.6 HMI / dashboard: small-medium (2–3 days)
- Enable the client stages (`m_DashState_NotAvailable`); show stage time and estimated time (present in the theory, missing in the code); per-client parameter screens.

### 2.7 Anybus (Siemens) integration: depends on the agreed spec
- Map CLIENT ID / CLIENT TYPE / RECIPE TYPE / REQUEST ON-OFF onto the queue, and return the CIP status. This part is on hold until the open interface questions are agreed.

### 2.8 Fix existing issues before extending (2–3 days)
- `bConcentration_correct` semantics, Auto → Manual on `init`, steps that always return TRUE (`Fill_Water_T94_Done`, `StartPump_CIP4`), CIP numbering in step names, the overweight check disabled by `AND FALSE` in MAIN.

## 3. Risks
- **Concurrency with production**: washing one tank or line while the neighbouring one produces needs safe valve separation. `ST_MixProof_state` exists in the HMI library, but no interlock logic was found.
- **No per-client state today**, so there is nothing to tell production "clean / not clean".
- **The sprinkler is "only for tanks"** (theory): it needs a separate parameter set and a flow split.
- **The volume-based steps** use tank level deltas. For long client lines, push-out and phase separation by volume alone may be insufficient without a conductivity sensor on the return.

## 4. Open questions (blocking the estimate)
1. Which clients does the **new** production have? Are they the same as `E_CIP_Usage` (Tank_1..4, TriBlender Marinade/Color, Line Marinade/Color) or different?
2. Client return: by gravity or by a return pump? Is there a client return pump (cf. `r_CIP_Client_Pump`)?
3. Sensors on the client return: is there conductivity for phase separation? Where are CT1101 / CT1101_2 / CT3101 / CT4101 / CT4102 and FT4001 / FT4002 located?
4. Are `S_*_Up` the tank sprinkler (spray ball) valves and `S_*_Low` the bottom/pipe valves?
5. Can clients be washed while other clients are in production? Which mixproof valves separate them?
6. The pipe volume of each client (for `r_Volume_Solution_Fill` and push-out).
7. Who starts a client wash: the HMI, Siemens (Anybus), or both? And who is the master of the "clean" status?
