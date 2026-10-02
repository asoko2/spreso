# syncPresence() flowchart

Source: `module/emp.presence/response/ProcessEmpPresence.proc.class.php`, `syncPresence()` (lines 113–1794).

The function turns raw machine taps (`emp_presence_time_log`) into one `emp_presence` row per employee per date, plus `in`/`out` rows in `emp_presence_time`. Each day takes one of six branches (A–F), depending on the employee's shift roster for that date.

Diagrams:

1. Overview: entry points, loops, branch dispatch, commit
2. Data flow: which tables are read and written
3. Branch A: day off (shift 3001)
4. Branches B and F: regular office hours, including overtime
5. Branches C, D, E: shift rosters (1–4 shifts per day)

---

## 1. Overview

```mermaid
flowchart TD
    subgraph ENTRY["Entry points"]
        C1["DoSyncEmpPresence.json<br/>manual Sync button<br/>is_daily = null"]
        C2["ViewSyncEmpPresenceDaily.html<br/>daily job: yesterday, every unit<br/>is_daily = 1"]
    end

    C1 --> S
    C2 --> S
    S(["syncPresence(start_date, end_date, unit_id, nip, is_daily)"]) --> TX["StartTrans()"]
    TX --> N{"nip given?"}
    N -->|yes| N1["log label = nip"]
    N -->|no| N2["getDetailUnit(unit_id)<br/>log label = unit name"]
    N1 --> DM{"is_daily == 1?"}
    N2 --> DM
    DM -->|yes| DM1["period = start_date<br/>period_end = end_date<br/>tanggal = start_date"]
    DM -->|no| DM2["period, period_end =<br/>start_date, end_date as Y-m-d"]
    DM1 --> LOG["INSERT emp_presence_log<br/>trans = 'sync_presence'"]
    DM2 --> LOG
    LOG --> EL{"is_daily == 1?"}
    EL -->|yes| EL1["getEmpPresenceSyncByDate<br/>employees active on tanggal"]
    EL -->|no| EL2["getEmpPresenceSync<br/>employees active in the period"]
    EL1 --> EMP
    EL2 --> EMP

    EMP[["for each employee"]] --> HO["cekUnitKerjaPusatOld, then cekUnitKerjaPusat<br/>is the unit head office (Kantor Pusat)?"]
    HO --> DEL["DELETE emp_presence_time<br/>trans = 'presence', whole period"]
    DEL --> DAY[["for each date in the period"]]

    DAY --> HS{"cekShiftByEmpIdDateRange<br/>shift roster on this date?"}
    HS -->|no| BF["F. No roster:<br/>regular office hours"]
    HS -->|yes| GS["getShiftByEmpId for<br/>today, tomorrow, yesterday"]
    GS --> Z{"shifts today?"}
    Z -->|none| NX
    Z -->|some| SW{"first shift_id,<br/>then shift count"}
    SW -->|"3001 (day off)"| BA["A. Day off (libur)"]
    SW -->|"3000 or 3100 (regular)"| BB["B. Regular shift"]
    SW -->|"2 shifts"| BC["C. Two shifts"]
    SW -->|"3 or 4 shifts"| BD["D. Three or four shifts"]
    SW -->|"1 other shift"| BE["E. Single shift"]
    SW -->|"5 or more"| NX

    BA --> NX
    BB --> NX
    BC --> NX
    BD --> NX
    BE --> NX
    BF --> NX

    NX{"more dates?"} -->|yes| DAY
    NX -->|no| NE{"more employees?"}
    NE -->|yes| EMP
    NE -->|no| AL["insertLog into gtfw_log_response<br/>'Sync Presensi, Unit Kerja ...'"]
    AL --> ET["EndTrans(result)<br/>commit if true, rollback if false"]
    ET --> MS["Messenger: success or fail message"]
    MS --> R(["return result"])

    classDef write fill:#fde2c8,stroke:#c46a1b,color:#3b1d00
    classDef branch fill:#dbe8ff,stroke:#3a63b8,color:#0b1f4d
    class LOG,DEL,AL write
    class BA,BB,BC,BD,BE,BF branch
```

---

## 2. Data flow (tables)

```mermaid
flowchart LR
    UP["import(): upload attlog file"] -->|"REPLACE INTO"| T2

    subgraph SRC["Read"]
        T1[("emp_employee, emp_employee_unit,<br/>emp_grade, emp_employee_type,<br/>emp_functional_position")]
        T2[("emp_presence_time_log<br/>raw machine taps")]
        T3[("emp_employee_shift_date,<br/>emp_employee_shift, emp_shift")]
        T4[("emp_shift_reguler<br/>office hours per weekday")]
        T5[("emp_day_off_week, emp_day_off_special,<br/>emp_day_off, emp_day_off_unit")]
        T6[("gtfw_unit, gtfw_unit_flexible")]
    end

    T1 -->|"emp_id, emp_unit_id,<br/>emp_access_code"| SP
    T2 -->|"earliest or latest tap<br/>in a time window"| SP
    T3 -->|"shifts for yesterday,<br/>today, tomorrow"| SP
    T4 -->|"start, end, hour_count,<br/>flexy_hour_count"| SP
    T5 -->|"weekly off, Ramadan,<br/>unit day off"| SP
    T6 -->|"head office?"| SP

    SP{{"syncPresence()"}}

    subgraph DST["Written"]
        W1[("emp_presence_log<br/>one row per run")]
        W2[("emp_presence<br/>one row per employee per date")]
        W3[("emp_presence_time<br/>'in' and 'out' rows")]
        W4[("gtfw_log_response<br/>activity log")]
    end

    SP -->|INSERT| W1
    SP -->|"INSERT or UPDATE: in, out, duration,<br/>balance, overtime (presence_balance_time)"| W2
    SP -->|"DELETE trans 'presence', then INSERT"| W3
    SP -->|INSERT| W4

    W2 --> RP["read by reports:<br/>emp.rep.presence, emp.rep.overtime"]
```

Machine taps are matched to employees by `emp_employee.emp_access_code = emp_presence_time_log.presencetimelog_nip`.

---

## 3. Branch A: day off (shift 3001)

```mermaid
flowchart TD
    A0(["A. shift_id = 3001"]) --> A1["DELETE emp_presence_time<br/>shift 3001 rows on this date"]
    A1 --> A2{"cekEmpPresence:<br/>emp_presence row for this date?"}
    A2 -->|no| A3["INSERT emp_presence<br/>no in or out, duration 0, balance 0"]
    A2 -->|yes| A4["UPDATE emp_presence<br/>in and out emptied, duration 0, balance 0"]
    A3 --> A5["INSERT emp_presence_time<br/>'in' and 'out', shift 3001, no time"]
    A4 --> A5
    A5 --> A6(["next date"])
```

---

## 4. Branches B and F: regular office hours

B is a roster day with shift 3000 or 3100. F is a day with no roster at all. Both run the same logic. The differences are listed under the diagram.

```mermaid
flowchart TD
    B0(["B. shift 3000 or 3100<br/>F. no roster"]) --> B1["DELETE emp_presence_time<br/>trans 'presence' on this date"]
    B1 --> B2["getHariKerja: calendar flags for the date<br/>libur_mingguan = weekly day off<br/>libur_special = Ramadan<br/>libur = unit day off"]
    B2 --> B3["IN = earliest tap 00:00:01 to 11:59:59<br/>OUT = latest tap 12:00:00 to 23:59:59"]
    B3 --> B4{"Ramadan?"}
    B4 -->|yes| B5["hours = Ramadan hours for this weekday"]
    B4 -->|no| B6{"office hours defined<br/>for this weekday?"}
    B6 -->|"no (Sat or Sun)"| SKIP(["skip date, nothing written"])
    B6 -->|yes| B7["hours = office hours for this weekday"]
    B5 --> B8{"IN or OUT found?"}
    B7 --> B8
    B8 -->|no| OT0
    B8 -->|yes| B9["duration = OUT − IN, in hours<br/>balance = abs(duration − hour_count)"]
    B9 --> B10{"emp_presence row<br/>for this date?"}
    B10 -->|no| B11["INSERT emp_presence"]
    B10 -->|yes| B12["UPDATE emp_presence"]
    B11 --> B13{"no 'in' time row yet<br/>and IN found?"}
    B12 --> B13
    B13 -->|yes| B14["INSERT emp_presence_time 'in'"]
    B13 -->|no| B15
    B14 --> B15{"no 'out' time row yet<br/>and OUT found?"}
    B15 -->|yes| B16["INSERT emp_presence_time 'out'"]
    B15 -->|no| OT0
    B16 --> OT0

    OT0{"IN and OUT both found?"} -->|no| DONE(["next date"])
    OT0 -->|yes| OT1{"Ramadan?"}
    OT1 -->|yes| OT2{"working day?<br/>not libur_mingguan, not libur"}
    OT2 -->|yes| OTa["OT start = office end + 1h<br/>(no flexi time in Ramadan)"]
    OT2 -->|no| OTb["overtime = OUT − IN<br/>(whole day counts)"]
    OT1 -->|no| OT3{"working day?"}
    OT3 -->|no| OTb
    OT3 -->|yes| OT4{"head office unit?"}
    OT4 -->|no| OTa2["OT start = office end + 1h"]
    OT4 -->|yes| OT5{"OUT − IN longer than<br/>flexy_hour_count?"}
    OT5 -->|no| DONE
    OT5 -->|yes| OTc["OT start = IN + flexi hours + 1h"]
    OTa --> OTd["overtime = abs(OUT − OT start)"]
    OTa2 --> OTd
    OTc --> OTd
    OTd --> OTw["UPDATE emp_presence<br/>presence_balance_time = overtime"]
    OTb --> OTw
    OTw --> DONE

    NB1["Note: in branch B, flexy_hour_count_minutes<br/>is never set, so OT start = IN + 1h"] -.- OTc
    NB2["Note: abs() makes leaving before<br/>OT start count as positive overtime"] -.- OTd

    classDef write fill:#fde2c8,stroke:#c46a1b,color:#3b1d00
    classDef note fill:#fff6bf,stroke:#b59b00,color:#3d3300,stroke-dasharray:4 3
    class B1,B11,B12,B14,B16,OTw write
    class NB1,NB2 note
```

B vs F:

| | B (shift 3000/3100) | F (no roster) |
|---|---|---|
| `presencetime_shift_id` written | 3000 or 3100 | NULL |
| `flexy_hour_count_minutes` | never set, so treated as 0 | `flexy_hour_count × 60` |
| Calendar flags come from | `$detail_shift_reguler[0]` | `$value_date` (loop over `getHariKerja`) |

---

## 5. Branches C, D, E: shift rosters

Part 1 shows how each shift's IN and OUT times are chosen. Part 2 shows how they are written.

```mermaid
flowchart TD
    S0(["C, D, E. shift roster day"]) --> W["Tap windows, per shift<br/>IN tap = earliest tap within shift_start ± 3h<br/>OUT tap = latest tap within shift_end ± 3h<br/>(OUT window moves to tomorrow for overnight shifts)"]

    W --> F1
    subgraph FIRST["IN time of the first shift"]
        F1{"yesterday's last shift ends inside<br/>today's first-shift start window?<br/>(start_awal to start_akhir, or ±1h at 00:00)"}
        F1 -->|yes| F2{"yesterday's last shift<br/>has an OUT tap?"}
        F2 -->|yes| F3["IN = scheduled shift_start<br/>(back-to-back, no new tap needed)"]
        F2 -->|no| F4["IN = IN tap"]
        F1 -->|no| F4
    end

    F3 --> L1
    F4 --> L1
    subgraph LAST["OUT time of the last shift"]
        L1{"tomorrow's first shift starts inside<br/>today's last-shift end window?<br/>(end_awal to end_akhir, or ±1h at 00:00)"}
        L1 -->|yes| L2{"tomorrow's first shift<br/>has an IN tap?"}
        L2 -->|yes| L3["OUT = scheduled shift_end"]
        L2 -->|no| L4["OUT = OUT tap"]
        L1 -->|no| L4
    end

    L3 --> K
    L4 --> K
    K{"branch"}
    K -->|"E. one shift"| E1["IN rule, OUT rule"]
    K -->|"C. two back-to-back shifts<br/>(shift 2 starts in shift 1 end window)"| C1["shift 1: IN rule, scheduled end<br/>shift 2: scheduled start, OUT rule"]
    K -->|"C. two separate shifts"| C2["shift 1: IN rule, its own OUT tap<br/>shift 2: its own IN tap, OUT rule"]
    K -->|"D. three or four shifts"| D1["first: IN rule, scheduled end<br/>middle: scheduled start and end<br/>last: scheduled start, OUT rule"]
```

```mermaid
flowchart TD
    S1(["IN and OUT chosen for each shift"]) --> UP
    UP[["for each shift of the day<br/>(E: runs once)"]] --> U1{"cekEmpPresence:<br/>emp_presence row for this date?"}
    U1 -->|no| U2["INSERT emp_presence<br/>shift_duration = sum of hour_count of all shifts"]
    U1 -->|yes| U3["UPDATE emp_presence"]
    U2 --> U4{"row existed before this write<br/>and presence_trans = 'presence'?"}
    U3 --> U4
    U4 -->|yes| U5["INSERT emp_presence_time<br/>'in' and 'out' with this shift_id"]
    U4 -->|no| U6
    U5 --> U6{"more shifts?"}
    U6 -->|yes| UP
    U6 -->|no| NX(["next date"])

    U4 -.- N1["Note: on the first sync of a date the row does not exist yet,<br/>so the first shift gets no in/out time rows.<br/>They appear only after a second sync."]
    U3 -.- N2["Note: emp_presence keeps the last write.<br/>C: in/out/duration describe shift 2 only.<br/>D, E: first IN to last OUT."]

    classDef write fill:#fde2c8,stroke:#c46a1b,color:#3b1d00
    classDef note fill:#fff6bf,stroke:#b59b00,color:#3d3300,stroke-dasharray:4 3
    class U2,U3,U5 write
    class N1,N2 note
```

---

## Notes from tracing the code

These are not drawn above, but they affect the data:

1. **Shift branches skip time rows on the first sync** (lines 795, 976, 1167, 1355, 1554). `$presence` is looked up before the insert, so for a new date it is null and `$presence['presence_trans'] == 'presence'` is false.
2. **Overtime uses `abs()`** (lines 382, 409, 420 and the same lines in the other branches). If OUT is earlier than OT start, the difference is still stored as positive overtime. Example: office end 16:00, OUT 16:00, non-head-office: OT start = 17:00, overtime = 1h. `emp.rep.overtime` reads this column directly.
3. **Branch B head office overtime** (lines 408, 599). `flexy_hour_count_minutes` is only computed in branch F (lines 1608, 1617), so B shifts OT start by 1h instead of flexi hours + 1h.
4. **Days with only one tap.** If only IN or only OUT is found, `strtotime(null)` returns `false` (0), so `duration = abs(OUT − IN)` is measured from Unix time 0, about 497,000 hours. `presence_balance` is affected the same way. What gets stored depends on the column type and SQL mode.
5. **`$result = ...` instead of `$result = $result && ...`** in branches A, B and F (for example lines 196, 310, 352, 388, 1646). A later success overwrites an earlier failure. ADOdb's `CompleteTrans` still rolls back on SQL errors, so the data stays safe, but the function can return `true` and show "success" for a sync that was rolled back.
6. **Four-shift days** (line 1221 onwards). Only index 3 (last) and index 1 (middle) have their own case. Index 2 falls into the first-shift `else`, so shift 3 is written with shift 1's times.
7. **Five or more shifts** in a day match no branch and are skipped silently.
