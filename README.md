# Restful-booker API Testing with Postman

API test suite for the [Restful-booker](https://restful-booker.herokuapp.com/apidoc/index.html) playground API, built with Postman. It covers a full **create → read → update → delete** booking flow, authentication with a token, and a negative test for an invalid token.

## Goals

- Practise API testing against a public API using only its documentation as the source of truth.
- Chain requests with variables (token, booking id) so the whole flow runs unattended in the Collection Runner.
- Record where the documentation and the real behaviour differ.

## Tech

- Postman (collection, environment, Collection Runner)
- JavaScript test scripts (`pm.test`, `pm.expect`, Chai assertions)
- Target API: `https://restful-booker.herokuapp.com`

## Repository structure

```
.
├── README.md
├── postman/
│   ├── Restful-Booker.postman_collection.json
│   └── Restful-Booker.postman_environment.json
└── docs/
    └── runner-result.png
```

## How to run

1. Import both files from `postman/` into Postman.
2. Select the **Restful Booker** environment (top right) and check that `base_url` is `https://restful-booker.herokuapp.com`.
3. Right-click the **CRUD Flow** folder → **Run folder** → Run.
4. Expected result: **21 / 21 tests passed**.

Run the folder as a whole, not request by request. The requests depend on each other (see below), and sending `Delete Booking` twice will fail because the booking no longer exists.

## Test flow (folder `CRUD Flow`)

The folder creates one booking, works on it, and deletes it, so each run leaves no data behind.

| # | Request | Purpose | Tests |
|---|---------|---------|:-----:|
| 1 | `Auth-Create Token` | Get a token and store it in the environment variable `token` | 2 |
| 2 | `Create Booking` | Create a booking and store its id in `createbookingid` | 5 |
| 3 | `Get Booking Created` | Read the booking back and compare every field | 2 |
| 4 | `Update Booking` (PUT) | Replace the whole booking (full payload, token required) | 2 |
| 5 | `Partial Update Booking` (PATCH) | Change `firstname` and `totalprice` only; other fields must stay unchanged | 3 |
| 6 | `Update Booking - Invalid Token` | Negative test: a wrong token must be rejected with 403 | 2 |
| 7 | `Delete Booking` | Delete the booking (expects 201 as shown in the docs) | 2 |
| 8 | `Get Delete Booking` | Confirm the booking is gone (404) | 3 |

Three read-only requests sit outside the folder because they do not depend on the flow: `Health Check`, `Get Booking IDs`, `Get Booking by ID`.

## Variables

| Variable | Scope | Set by | Used by |
|----------|-------|--------|---------|
| `base_url` | Environment | Manual | All requests |
| `token` | Environment (current value) | `Auth-Create Token` | PUT, PATCH, DELETE via header `Cookie: token={{token}}` |
| `createbookingid` | Collection | `Create Booking` | Steps 3 to 8 |
| `bookingid` | Collection | `Get Booking IDs` | `Get Booking by ID` |

`createbookingid` and `bookingid` are separate on purpose. The first holds a booking this suite created and may modify or delete. The second holds an existing booking from the shared server and is only read.

## Design decisions

- **Tests follow the documentation.** An expected status or body is asserted only if the docs state it or if I observed it, and observed behaviour is labelled as such below.
- **Test data is created by the suite.** The API is shared by many users, so existing bookings can change or disappear at any time. Only the booking created in step 2 is modified.
- **Variables are written inside the test** that validates the value, right after the `expect`. If the response is wrong, the variable is not overwritten with a bad value.
- **Every new test was made to fail once** (by changing an expected value) before being kept, to confirm it can detect a real defect.
- **The token is stored in the environment's current value only**, not in the shared value, so it is not synced or exported by accident.

## Observations: documentation vs. real behaviour

These are discrepancies and undocumented behaviours found while building the suite. They are observations, not verdicts: the API is a playground and may be intentionally imperfect.

### Documentation inconsistencies

1. **DELETE status code.** The section heading says `Success 200`, but the description and example response say `201 Created`. The real response is `201 Created`.
2. **Optional fields are unclear.** The docs mark every PATCH body field as `optional`, but mark nothing as optional in the Create and PUT bodies, so it is not clear whether `additionalneeds` is required.
3. **Nested object missing in the field table.** In the Update request body, `checkin` and `checkout` are listed without their parent `bookingdates`.

### Undocumented behaviour (observed)

4. **Invalid token:** `PUT /booking/{id}` with a wrong token returns `403 Forbidden`. The body is plain text `Forbidden`, not JSON. The docs do not describe this case.
5. **Successful DELETE body:** the body is the plain text `Created`.
6. **Missing booking:** `GET /booking/{id}` for a deleted or unknown id returns `404 Not Found` with `Content-Type: text/plain` and the body `Not Found`.
7. **Response shape differs between endpoints:** `POST /booking` returns the booking wrapped in `{ "bookingid": ..., "booking": {...} }`, while `GET`, `PUT` and `PATCH` return the booking object directly.
8. **A new token is issued on every call** to `/auth`.

## Limitations and next steps

- The suite depends on a public, shared server, so it can fail if the server is down or very slow.
- Possible extensions: `PUT` with missing fields, `POST` with invalid data, duplicate or boundary values, running the collection from the command line with Newman, and running it automatically with GitHub Actions.

## Author

Johnny, QA / Tester fresher.
