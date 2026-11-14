# UnQue 2.0 - College appointment system (Node.js + Express + Amazon DocumentDB)

Backend APIs that let professors publish availability and students book appointments.
Deployed on **AWS EC2**, storing data in **Amazon DocumentDB** (MongoDB-compatible).

## What changed from the original

| Change | Where |
| --- | --- |
| **New API** - professor updates a free slot's timing without deleting it | `controllers/proff.js` (`updateAvailability`), `routes/proff.js` |
| DocumentDB-ready connection (TLS, `retryWrites=false`), fail-fast startup checks, `/health` route | `index.js`, `.env.example` |
| `GET .../availability` now also returns each slot's `slotId` (needed by the new API) | `controllers/general.js` |
| Errors now return their real HTTP status (they were all sent as `200`) | `middleware/error.js` |
| Cancelled status written as `cancelled` to match the model enum (was `canceled`) | `controllers/proff.js` |
| `start` script added, `dev` script fixed (`server.js` -> `index.js`), removed unused `bcrypt`, declared `body-parser` | `package.json` |
| EC2 setup script, Postman collection | `deploy/setup-ec2.sh`, `postman/` |

## API reference

Send the JWT from login as `Authorization: Bearer <token>`.

| Method | Path | Who | Purpose |
| --- | --- | --- | --- |
| POST | `/users/register` | anyone | body: `name, email, password, confirmPassword, role` (`student`/`professor`) |
| POST | `/users/login` | anyone | body: `email, password` -> `token, id, name, role` |
| GET | `/users/profile/:id` | anyone | user profile |
| POST | `/professor/availability` | professor | body: `startTime, endTime` (ISO dates) -> add a free slot |
| **PUT** | **`/professor/availability/:slotId`** | professor | **NEW** body: `startTime, endTime` -> change the timing of an existing free slot |
| PATCH | `/professor/cancel-appointments/:studentId` | professor | cancel that student's booked appointments |
| GET | `/general/professor/:professorId/availability` | logged in | list a professor's slots (with `slotId`) |
| POST | `/general/book` | logged in | body: `professorId, time` -> book an appointment |
| GET | `/general/appointments` | logged in | the student's booked appointments (404 if none) |
| GET | `/health` | anyone | server + DB status |

### Rules of the new API (`PUT /professor/availability/:slotId`)
- The slot document is **updated in place** (same `_id`) - it is never removed.
- Only the professor who owns the slot can change it (`404` otherwise).
- Only a **free** slot can change: if a booked appointment falls inside the slot -> `409`. (After the appointment is cancelled the slot is free again.)
- The new timing must be valid, `startTime < endTime`, and must not overlap another slot of the same professor (`400`).

## Environment variables

Copy `.env.example` to `.env`:

| Variable | Meaning |
| --- | --- |
| `PORT` | port to listen on (`3000` on EC2) |
| `JWT_SECRET` | secret used to sign tokens |
| `MONGO_URL` | MongoDB / DocumentDB connection string (see `.env.example`) |

## Run locally

```bash
npm install
cp .env.example .env   # set MONGO_URL=mongodb://localhost:27017/unque
npm start
```

## Deploy on AWS (EC2 + DocumentDB)

1. Security groups (same VPC): `ec2-sg` allows SSH (22) from your IP and TCP 3000 from anywhere; `docdb-sg` allows TCP 27017 **only from `ec2-sg`**.
2. Create an Amazon DocumentDB cluster (attach `docdb-sg`, TLS on) and note the cluster endpoint.
3. Launch an EC2 instance (Amazon Linux 2023, attach `ec2-sg`) and SSH in.
4. On the server:
   ```bash
   curl -O https://raw.githubusercontent.com/<you>/<repo>/main/deploy/setup-ec2.sh
   bash setup-ec2.sh https://github.com/<you>/<repo>.git
   ```
   The script installs Node 20/git/pm2/mongosh, downloads the DocumentDB CA bundle, writes `.env`, and starts the app with pm2.
5. Verify: `curl http://<EC2_PUBLIC_IP>:3000/health` -> `{"status":"ok","db":"connected"}`.
6. Import `postman/UnQue-EC2.postman_collection.json`, set `baseUrl` to `http://<EC2_PUBLIC_IP>:3000`, and run the flow.

Inspect the data from EC2:
```bash
mongosh --tls --host <cluster-endpoint>:27017 --tlsCAFile ~/global-bundle.pem \
  --retryWrites=false --username <user> --password
use unque
db.availabilities.find().pretty()
db.appointments.find().pretty()
```
