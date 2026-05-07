# Bug Report — Library Management System

**Date:** 2026-05-07  
**Tested against:** `http://localhost:8088`  
**Auth:** `admin` / `admin` (in-memory Spring Security)  
**Total requests fired:** 32 (edge-case / validation probes)  
**Errors logged:** 20 &nbsp;|&nbsp; Silent bugs (passed with bad data): 8 &nbsp;|&nbsp; Clean passes: 4

---

## Summary Table

| # | Endpoint | Input | HTTP Status | Type | Severity |
|---|---|---|---|---|---|
| 1 | `GET /delete-book` | `id=abc` | 400 | Type mismatch | Medium |
| 2 | `POST /edit-book` | `id=abc` | 400 | Type mismatch | Medium |
| 3 | `POST /add-book` | `book_price=notANumber` | 400 | Type mismatch | Medium |
| 4 | `GET /delete-book` | *(no id param)* | **500** | Missing param → unhandled | **High** |
| 5 | `GET /delete-member` | *(no id param)* | **500** | Missing param → unhandled | **High** |
| 6 | `GET /get-book-details-one` | *(no book_name)* | 400 | Missing param | Medium |
| 7 | `POST /edit-book` | *(no bookname)* | 400 | Missing param | Medium |
| 8 | `GET /get-member-details-one` | *(no member_name)* | 400 | Missing param | Medium |
| 9 | `POST /edit-member` | *(no memberage)* | 400 | Missing param | Medium |
| 10 | `GET /set-returndate` | *(no member)* | 400 | Missing param | Medium |
| 11 | `GET /del-smartbs` | *(no name)* | 400 | Missing param | Medium |
| 12 | `POST /sml-main` | *(no bookName2/3)* | 400 | Missing param | Medium |
| 13 | `POST /add-book` | `book_price=-99.99` | 400 | Negative price | Medium |
| 14 | `POST /add-book` | *(all fields empty)* | 400 | Empty fields | Medium |
| 15 | `POST /add-book` | `book_name=AAA…` (250 chars) | 400 | No length validation | Low |
| 16 | `GET /delete-member` | `id=abc` | 400 | Type mismatch | Medium |
| 17 | `POST /edit-member` | `memberage=notAnInt` | 400 | Type mismatch | Medium |
| 18 | `POST /add-member` | `member_phone_number=99999999999999999999` | 400 | Long overflow | Medium |
| 19 | `POST /add-member` | *(all fields empty)* | 400 | Empty fields | Medium |
| 20 | `POST /sml-main` | *(no memberName)* | 400 | Missing param | Medium |

---

## Silent Bugs (200 / 302 — bad data accepted)

These requests returned a success status but stored or processed invalid data with no error or validation message.

| # | Endpoint | Input | Status | Bug |
|---|---|---|---|---|
| 1 | `GET /delete-book` | `id=-1` | 302 | Negative ID silently ignored — no validation |
| 2 | `GET /delete-book` | `id=99999` | 302 | Non-existent ID silently ignored — no validation |
| 3 | `POST /edit-book` | `bookname=` | 302 | Book name set to empty string — accepted without error |
| 4 | `POST /add-member` | `member_age=-5` | 302 | Negative age stored — no validation |
| 5 | `POST /sml-main` | `bookName=C&bookName2=C&bookName3=C` | 200 | Same book borrowed 3 times — no duplicate check |
| 6 | `POST /sml-main` | `memberName=GhostMember` | 200 | Non-existent member creates a borrow record — no existence check |
| 7 | `GET /del-smartbs` | `name=GhostMember` | 302 | Ghost member deletion silently ignored |
| 8 | `GET /set-returndate` | `member=GhostMember` | 302 | Return date set for unknown member — `checkMember` returns `null` silently |

---

## Detailed Bug Notes

### BUG-01 — 500 on missing `id` for delete endpoints (High)

**Endpoints:** `GET /delete-book`, `GET /delete-member`  
**Root cause:** Both controllers declare `id` as a raw `int` method parameter without `@RequestParam`. When `id` is absent from the query string, Spring cannot bind it and throws an unhandled exception, resulting in a 500 instead of a proper 400.

```java
// BookController.java
public String deleteBookFromList(int id) { ... }

// MemberController.java
public String deleteMemberFromList(int id) { ... }
```

**Fix:** Annotate with `@RequestParam` and provide a default, or add proper validation.

---

### BUG-02 — Error message always shown on book detail page (Medium)

**Endpoint:** `GET /get-book-details-one?book_name=C`  
**Root cause:** `BookController` unconditionally puts the error string into the model, even when the book is found. `MemberController` correctly guards this with a null check — `BookController` does not.

```java
// BookController.java — error always added regardless of result
model.put("clickbook", services.getBookDetails(book_name));
{
    model.put("error", "No Subjects are available"); // ← always runs
}

// MemberController.java — correct pattern
if ((services.getMemberDetails(member_name)) == null) {
    model.put("error", "No Subjects are available");
}
```

---

### BUG-03 — No input validation on any entity (Medium / Low)

No `@Valid`, `@NotBlank`, `@Min`, `@Max`, or `@Size` annotations are present on any POJO or controller parameter. This allows:

- Negative book prices (`book_price=-99.99`)
- Negative member ages (`member_age=-5`)
- Empty book/member names (`book_name=`, `member_name=`)
- Arbitrarily long strings stored without truncation

---

### BUG-04 — Phone number overflow (Medium)

**Endpoint:** `POST /add-member`  
**Input:** `member_phone_number=99999999999999999999`  
**Root cause:** `member_phone_number` is typed as `long`. A value exceeding `Long.MAX_VALUE` (9,223,372,036,854,775,807) causes a binding error (400). However there is no upper/lower bound validation — any value within `long` range, including negatives, is accepted silently.

---

### BUG-05 — Borrow system has no existence or duplicate checks (Medium)

**Endpoint:** `POST /sml-main`  
- A non-existent member name creates a borrow record with no validation against the member list.
- The same book can be passed as `bookName`, `bookName2`, and `bookName3` — no duplicate check.
- `bookName2` and `bookName3` have no `required=false` default, so omitting them throws a 400 — callers must always pass all three, even if empty.

---

### BUG-06 — `MemberController.getBookDetails` calls `getMemberDetails` twice (Low)

```java
// MemberController.java
model.put("clickmember", services.getMemberDetails(member_name));  // call 1
if ((services.getMemberDetails(member_name)) == null) {            // call 2 — redundant
    model.put("error", "No Subjects are available");
}
```

The result is computed twice. Should be stored in a local variable first.

---

---

## Log Confirmation

The application log was cross-referenced against all 32 test requests.

### All 20 errors are present in the log

| Exception type in log | Count | Covers |
|---|---|---|
| `MissingServletRequestParameterException` | 8 | Missing `book_name`, `bookname`, `bookName2`, `memberName`, `member_name`, `memberage`, `member`, `name` |
| `NumberFormatException` | 16 | `"abc"` (×6), `"notAnInt"` (×2), `"notANumber"` (×2), `""` empty fields (×4), `"99999999999999999999"` overflow (×2) |
| `MethodArgumentTypeMismatchException` | 4 | String→int on `id` (×3) and `memberage` (×1) |
| `IllegalStateException` (→ 500) | 2 | `delete-book` and `delete-member` with no `id` — primitive `int` cannot bind to null |
| `BindException` / `ConversionFailedException` | 6+8 | `book_price=notANumber`, empty required fields on `BookPojo`/`MemberPojo` |

Exact log message for the two 500s (BUG-01):
```
Optional int parameter 'id' is present but cannot be translated into a null value
due to being declared as a primitive type. Consider declaring it as object wrapper
for the corresponding primitive type.
```

### The 8 silent bugs leave **zero** error lines in the log

| Request | Log lines | Error lines |
|---|---|---|
| `delete-book?id=-1` | 1 (DEBUG only) | 0 |
| `delete-book?id=99999` | 1 (DEBUG only) | 0 |
| `edit-book` empty bookname | 0 (params masked) | 0 |
| `add-member member_age=-5` | 0 (params masked) | 0 |
| `sml-main` GhostMember | 3 (DEBUG only) | 0 |
| `sml-main` duplicate books | 0 (params masked) | 0 |

These are the most dangerous category: invalid data is accepted, stored, and the log gives no indication anything went wrong.

---

## Security Audit Result

✅ **SAFE TO RUN** — Full audit found no malicious code, backdoors, dangerous shell execution, outbound network calls, or supply-chain issues. See separate security findings for details.

Hardcoded credentials (`admin`/`admin`) are expected for a demo application and pose no risk in a local development context.
