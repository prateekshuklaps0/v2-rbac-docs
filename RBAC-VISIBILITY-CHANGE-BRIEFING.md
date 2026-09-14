# Lead Visibility Change — Who Is Affected and Why

**Date:** 2026-08-06
**Scope:** Organisation 12 (Masters' Union), admin portal — Leads, Applicants, Dashboard, Calendar, Exports
**Status:** Live

---

## 1. In one paragraph

Lead visibility now follows the role hierarchy. Previously, almost anyone who logged into the admin portal saw **every lead in the organisation**, because the system decided "is this an admin?" by reading a leftover column from the v1 app that says which *portal* you log into — not how senior you are. Every Team Lead, Manager, GM and Sales Head carried the value `admin` in that column, so all of them were treated as full organisation owners.

The system now uses the **role level** instead. A user sees their own records, their reporting line, and everyone at a level below theirs. Only level-6 Admins see everything.

**This is a correction, not a data loss. No leads were deleted, moved, or reassigned.**

---

## 2. The hierarchy now being enforced

| Level | Role | Sees |
|---:|---|---|
| 6 | Admin | Everything in the organisation |
| 5 | Sales Head, and the level-5 admin roles | Themselves + GM, Manager, Team Lead, Counsellor |
| 4 | GM | Themselves + Manager, Team Lead, Counsellor |
| 3 | Manager | Themselves + Team Lead, Counsellor |
| 2 | Team Lead | Themselves + Counsellor |
| 1 | Counsellor | Themselves only |

**Peers are not visible to each other.** A Team Lead cannot see another Team Lead's leads. This was confirmed with the POC and the reporting manager before release.

---

## 3. Where the numbers went

Total active leads in the org: **426,045**

| Bucket | Count | Who can see it now |
|---|---:|---|
| Owned by someone below level 5 | 201,257 | Level 5 and above |
| **Unassigned** (no counsellor) | **146,334** | **Level 6 Admins only** |
| Owned by a level-5 or level-6 user | 44,035 | That person, and level 6 |
| Owned by a user with no role in org 12 | 34,419 | Level 6 Admins only |

A level-5 user therefore now sees roughly **201,257 leads plus any leads assigned to them personally**, where before they saw all 426,045.

> **The 146,334 unassigned leads are the item needing a business decision.** They were never visible to counsellors (an unassigned lead matches nobody), but they *were* visible to the 98 non-counsellor users who previously bypassed scoping. Now only 18 Admins can see them. If anyone below level 6 is responsible for distributing incoming leads, that workflow needs either a rule change or a level change.

---

## 4. Users affected — level 5

All 32 users below lose organisation-wide visibility and now see the level-5 view described above.

### Real staff accounts

| Role | Name | Email | Logins | Own leads kept |
|---|---|---|---:|---:|
| ADMIN 2 | Maninder Singh | maninder.singh@venturepact.com | 66 | 0 |
| ADMIN 2 | Khushboo DEV | khushboo.dungriyal@mastersunion.org | 15 | 0 |
| ADMIN 2 | Dinky | dinky.jain@mastersunion.org | 12 | 0 |
| ADMIN 2 | Naman | naman.kareer@mastersunion.org | 5 | 0 |
| ADMIN 2 | Ujjwal Sood | ujjwal.sood+77@mastersunion.org | 3 | 0 |
| ADMIN 2 | Karan Dev | karan.chauhan+v2@mastersunion.org | 2 | 0 |
| ADMIN 2 | Shilpa DEV | shilpa.bijalwan@mastersunion.org | 1 | 0 |
| ADMIN 2 | Ujjwal Sood *(disabled)* | ujjwal.sood@mastersunion.org | 2 | 0 |
| PGP ADMIN | **Kishan** | kishan.soni@mastersunion.org | 46 | **28,822** |
| PGP ADMIN | Kuldeep Rawat | kuldeep.rawat@mastersunion.org | 44 | 0 |
| PGP ADMIN | Samarth Bhagtani | samarth.bhagtani@mastersunion.org | 7 | 0 |
| PGP ADMIN | Naman Satija | naman.satija@mastersunion.org | 1 | 0 |
| PGP ADMIN | Naveen Sagar | naveen.sagar@mastersunion.org | 1 | 0 |
| Executive ADMIN | Arun Arya | arun.arya@mastersunion.org | 30 | 61 |
| Executive ADMIN | Karan Singh | karan.singh@mastersunion.org | 4 | 1 |
| Executive ADMIN | Nikhil Mittal | nikhil.mittal@mastersunion.org | 1 | 0 |
| Executive ADMIN | Naman Satija | naman.satija@mastersunion.org | 1 | 0 |
| Executive ADMIN | Naveen Sagar | naveen.sagar@mastersunion.org | 1 | 0 |
| Sales Head | **Vishwanath Nair** | vishwanath.nair@mastersunion.org | 4 | **14,089** |
| Sales Head | Ratan Anmol Sethi | ratan.sethi@mastersunion.org | 3 | 0 |
| Sales Head | Mrinal Kashyap | mrinal.kashyap@mastersunion.org | 2 | 0 |
| Sales Head | **Vikas Singha** | vikas.singha@mastersunion.org | 0 | **1,062** |

*Naman Satija and Naveen Sagar hold both Executive ADMIN and PGP ADMIN. Both are level 5, so their view is the same either way.*

### Test / non-staff accounts (no action expected)

| Role | Name | Email | Logins |
|---|---|---|---:|
| Test Role | Test Admin fourty nine | test@admin49.com | 9 |
| Test Role | mmflc | mmflc84476@minitts.net | 0 |
| Test Role | test | shfkfi9513@minitts.net | 0 |
| Test Role | TEST | hevad17744@luxudata.com | 0 |
| Test Role | Test admin three | test@61.com | 0 |
| Test Role | ucevb | ucevb49201@minitts.net | 0 |
| Visitor | sopeve | sopeve2903@lasttea.com | 6 |
| Visitor | Devansh Bharadwaj | devansh.bharadwaj@tetr.org | 0 |
| ADMIN 2 | TESTR ADMIN | sharmila.saren+admin2@mastersunion.org | 2 |
| ADMIN 2 | Maninder Singh | 5r1opki68m@gmeenramy.com | 1 |

---

## 5. Also affected

### Levels 4, 3 and 2 — 62 more users

GM (13), Manager (14), Team Lead (31) and the smaller level-2 roles previously saw everything too. They now see their own level's view. Same cause, same correction.

### Three accounts will see nothing at all

| Role | Name | Email |
|---|---|---|
| Admin View Only | Shagun Garg | shagun.garg@mastersunion.org |
| Executive View | Akshay Saluja | akshay.saluja@mastersunion.org |
| Executive View | Sharmila Saren (test) | sharmila.saren+xx6@mastersunion.org |

These roles sit at **level 1** with no leads assigned and no reports, so there is nothing below them and nothing of their own.

**This is a role configuration issue, not a bug.** The roles were given the lowest level to stop them editing — but level controls *how much you can see*, while a separate permission list controls *what you can do*. A read-only role should be given a **high level** with **view-only permissions**. Raising the level does not grant edit rights.

### 34,419 leads sitting on unusable accounts

Three accounts hold leads but have no role, no organisation membership, and have **never been logged into**. Each is a duplicate of a real person's working account:

| Ghost account | Leads | The person's real account |
|---|---:|---|
| shrashti.takrani@mastersunion.org (id 4627799) | 26,789 | same email, id 4627749 — 8 logins, has a role |
| ishan.ali1+7@mastersunion.org | 3,747 | ishan.ali1@mastersunion.org — 7 logins, PGP TBM Counsellors |
| sunny.singh+8@mastersunion.org | 3,709 | sunny.singh@mastersunion.org — has a role |

These leads are live — some were updated the day before this change. They are now visible only to level-6 Admins, and the person they belong to still cannot see them, because they are attached to the unused duplicate rather than the account that person signs into.

**Recommended:** reassign these leads to each person's real account. This is data cleanup, not a code change. The duplicates were created in Aug 2025 and Jul 2026 and are unrelated to this release.

---

## 6. What to do

| # | Action | Owner |
|---|---|---|
| 1 | Decide who should see the **146,334 unassigned leads**. If someone below level 6 distributes leads, they need either a level change or a rule change. | Business |
| 2 | Confirm whether `ADMIN 2`, `PGP ADMIN` and `Executive ADMIN` are meant to be **full** admins. If yes, move them to level 6 — do not revert the change. | Business + Tech |
| 3 | Fix the three view-only roles by raising their level, keeping view-only permissions. | Tech |
| 4 | Reassign the 34,419 leads from the three duplicate accounts to the real ones. | Data / Ops |
| 5 | Consider a unique-email guard on staff accounts to stop duplicates recurring (requires cleanup first). | Tech |

---

## 7. Notes for whoever picks this up

- Any export queued **before** this release fails with `roleId (the acting role) is required`. Re-running it works.
- If a user reports `400 roleId required` on the leads page, they should log out and back in.
- The **Queries** and **Communication** dashboard tiles are still organisation-wide — they have no link to a lead owner, so they cannot be scoped yet. Expect them to look inconsistent beside the scoped tiles.
- Email campaigns (`campaignRecipientResolver`) are still organisation-wide and are not affected by this change.
- Full technical detail: `v2-rbac-docs/USERS_MODULE_ANALYSIS.md` §15, and the documentation block above `listUsersManagedByUser` in `be-anandi/src/v2/services/userService.js`.
