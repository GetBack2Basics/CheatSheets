# User Profiles & Project Assignments Playbook

> **Context:** This playbook captures the pattern used in SpatialCourse_Crafter (FunGIS Spatial Olympics Platform) for managing users, roles, teams, and course assignments. It generalizes the RBAC approach so the same pattern can be applied to any similar platform.

---

## Role Hierarchy

| Role | Who sets it | Capabilities |
|---|---|---|
| `SUPER_ADMIN` | Hard-coded to one primary email | Full control: manage all users, assign/revoke admin roles, edit any course |
| `ADMIN` | Granted by Super Admin | Create/edit teams, assign courses to teams, edit own courses |
| `PLAYER` | Default for new sign-ups | Access only courses assigned to their team |

> **Rule of thumb:** Hardcode exactly one Super Admin email at startup. All other role elevation is done at runtime through the Admin UI, never by editing source.

---

## Data Model

### User Record

```js
{
  email: "user@example.com",       // Primary key (lowercase, trimmed)
  name: "Full Name",
  role: "SUPER_ADMIN" | "ADMIN" | "PLAYER",
  assignedTeamIds: ["team-abc"],   // Multiple teams supported
  teamId: "team-abc",              // Primary (first) team (convenience field)
  assignedCourseIds: ["course-xyz"],
  organization: "",
  phone: "",
  notes: "",
  authProvider: "EMAIL" | "GOOGLE",
  picture: "",
  createdAt: "ISO8601",
  updatedAt: "ISO8601"
}
```

### Team Record

```js
{
  id: "team-<timestamp>",          // Generated: `team-${Date.now()}`
  name: "Team Display Name",
  members: ["email1@x.com", "email2@x.com"],  // Array of lowercase emails
  assignedCourseIds: ["course-xyz", "course-abc"]
}
```

### Course Record (abbreviated)

```js
{
  id: "course-<slug>",
  title: "Course Title",
  createdBy: "admin@example.com",  // Used for permission guard on edit
  clues: [],
  // ... other course fields
}
```

---

## Persistence Strategy

The system uses a **dual-layer** persistence approach:

1. **Optimistic local cache** — `localStorage` is read first on startup, so the UI is instantly populated.
2. **Async cloud sync** — An async `fetch('/api/users')` fires after the initial render. If the server returns data, it overwrites the local cache and triggers a re-render via subscribers.

```js
// Pattern: optimistic local load → async cloud override
function loadUsers() {
  let localUsers = INITIAL_USERS;  // Always start with known defaults
  try {
    const stored = localStorage.getItem(USERS_STORAGE_KEY);
    if (stored) localUsers = JSON.parse(stored);
  } catch (e) { /* silent */ }

  // Async cloud sync — overwrites local on success
  fetch('/api/users')
    .then(res => res.json())
    .then(data => {
      if (data.success && data.users.length > 0) {
        this.users = data.users;
        localStorage.setItem(USERS_STORAGE_KEY, JSON.stringify(data.users));
        this.notify();
      }
    })
    .catch(err => console.warn('Cloud sync failed:', err));

  return localUsers;
}
```

> **Server backend:** The server stores all arrays (`users`, `teams`, `courses`, `submissions`) in a single `db.json` file. Each `POST` replaces the full array. Simple and predictable for small-scale deployments.

```js
// Server-side: GET and POST full arrays
app.get('/api/users',  (req, res) => res.json({ success: true, users: getDB().users }));
app.post('/api/users', (req, res) => {
  const db = getDB();
  db.users = req.body;   // Full array replacement
  saveDB(db);
  res.json({ success: true, users: db.users });
});
```

---

## Authentication Flow

The app supports three sign-in methods:

```
┌────────────────────────────────────────┐
│           Sign-In Options              │
│                                        │
│  1. Email + 6-digit code               │
│     POST /api/auth/send-code           │
│     POST /api/auth/verify-code         │
│                                        │
│  2. Google OAuth                       │
│     POST /api/auth/google              │
│                                        │
│  3. Direct email (no password)         │
│     authService.signIn(email)          │
└────────────────────────────────────────┘
```

### New User Auto-Creation

When a user signs in for the first time (any method), the system:
1. Checks if the email already exists in `this.users`
2. If not found → creates a new record with `role: 'PLAYER'` (unless it's the Super Admin email)
3. Saves to both `localStorage` and cloud `/api/users`

```js
if (!existing) {
  const role = cleanEmail === SUPER_ADMIN_EMAIL ? 'SUPER_ADMIN' : 'PLAYER';
  existing = { email: cleanEmail, name, role, createdAt: new Date().toISOString() };
  this.saveUsers([...this.users, existing]);
}
this.saveSession(existing);
```

---

## Admin: Creating a User Profile

Admins do **not** need to pre-create users — they appear automatically on first sign-in. However, admins can pre-populate a user's profile (name, team, course assignments) via `updateUserProfile`.

### Steps via Admin UI

1. Navigate to **Admin Panel → User Management**
2. Click **Add User** (or search for an existing one by email)
3. Fill in:
   - **Email** — must be valid (this becomes the primary key)
   - **Display Name**
   - **Role** — only Super Admin can set `ADMIN`; Admins can only manage `PLAYER`
   - **Assigned Teams** — select one or more from the team list
4. Save — the user record is synced to the server

### Programmatic (authService API)

```js
// Update existing user profile (requires ADMIN or SUPER_ADMIN session)
authService.updateUserProfile('user@example.com', {
  name: 'Jane Smith',
  role: 'ADMIN',          // Only Super Admin can set this
  assignedTeamIds: ['team-1234567890'],
  assignedCourseIds: ['course-cairns-survey'],
  organization: 'FunGIS',
  phone: '+61 400 000 000',
  notes: 'Field coordinator'
});
```

> **Key rule:** Changing a user's `assignedTeamIds` also syncs the `members` array of the affected teams — no manual team update required.

---

## Admin: Managing Teams

### Create a Team

```js
// Requires: isAdmin() === true
authService.createTeam(
  'North Coast GIS Team',              // Team display name
  ['alice@x.com', 'bob@x.com'],        // Initial member emails
  ['course-coastal-survey']            // Pre-assign course IDs
);
// Returns: { id: 'team-1234567890', name: '...', members: [...], assignedCourseIds: [...] }
```

### Update a Team

```js
authService.updateTeam('team-1234567890', {
  name: 'Updated Team Name',
  members: ['alice@x.com', 'charlie@x.com'],  // Replaces the full member list
  assignedCourseIds: ['course-a', 'course-b']
});
// Side-effect: updates assignedTeamIds on each affected User record
```

### Add a Single User to a Team

```js
authService.assignUserToTeam('bob@x.com', 'team-1234567890');
```

### Delete a Team

```js
authService.deleteTeam('team-1234567890');
// Does NOT remove the team from existing user.assignedTeamIds — clean up manually if needed
```

---

## Admin: Assigning Projects (Courses) to Teams

Courses are assigned to **teams**, not individuals. Players get access through their team membership.

```js
// Assign a course to one or more teams (replaces previous assignments for that course)
authService.assignCourseToTeams('course-cairns-hilton-survey', [
  'team-1234567890',
  'team-9876543210'
]);
// Teams NOT in the teamIds array will have this course removed from their assignedCourseIds
```

> **Access check:** At runtime, the Player UI filters the visible course list to only those whose ID appears in the player's team's `assignedCourseIds`.

### Permission Guards

```js
// Admins can only edit courses they created
authService.canEditCourse(course);
// → true if SUPER_ADMIN, or if ADMIN and course.createdBy === currentUser.email

// Role check helpers
authService.isSuperAdmin();   // true only for the hardcoded Super Admin email
authService.isAdmin();        // true for SUPER_ADMIN or ADMIN roles
```

---

## Super Admin: Role Escalation

Only the Super Admin can promote/demote other users:

```js
authService.setRole('target@example.com', 'ADMIN');   // Promote to Admin
authService.setRole('target@example.com', 'PLAYER');  // Demote to Player
// Note: Cannot alter the Super Admin's own role via this method
```

---

## Reactivity Pattern

The `AuthService` uses a simple pub/sub observer pattern. Any component that needs to react to user/team changes subscribes:

```jsx
// In a React component:
useEffect(() => {
  return authService.subscribe(({ currentUser, users, teams }) => {
    setCurrentUser(currentUser);
    setUsers(users);
    setTeams(teams);
  });
}, []);

// Or to just trigger a re-render on auth changes:
const [, setAuthTick] = useState(0);
useEffect(() => authService.subscribe(() => setAuthTick(t => t + 1)), []);
```

---

## End-to-End Playbook: Onboarding a New Participant

```
1. Admin signs in
   └─ Opens Admin Panel → User Management

2. Create / confirm team exists
   └─ authService.createTeam('Team Name', [], [])
      OR verify in team list

3. Assign the relevant course to the team
   └─ authService.assignCourseToTeams('course-id', ['team-id'])

4. Participant signs in (any method)
   └─ Auto-created as PLAYER with email as primary key

5. Admin updates participant's profile
   └─ authService.updateUserProfile('participant@x.com', {
        name: 'Jane Smith',
        assignedTeamIds: ['team-id']
      })
   └─ Team members array is automatically synced

6. Participant refreshes
   └─ Cloud sync fires, loads updated user + team data
   └─ Course list filtered to only their team's assigned courses
   └─ Ready to play!
```

---

## Common Gotchas

| Gotcha | Fix |
|---|---|
| User logged in but sees no courses | Check their `teamId` is set and the team has `assignedCourseIds` populated |
| Role change not taking effect | User must sign out and back in (session is loaded from `localStorage` on mount) |
| Team membership out of sync | Use `updateUserProfile` (not `updateTeam`) when changing a user — it syncs both sides |
| Super Admin email changed in code | Clear `localStorage` on all clients or they'll retain old role from cached session |
| `POST /api/users` replaces full array | Always pass the complete user list; never send a single user object |

---

## Environment Variables (Server)

| Variable | Purpose |
|---|---|
| `SMTP_HOST` / `EMAIL_HOST` | SMTP server hostname |
| `SMTP_PORT` / `EMAIL_PORT` | SMTP port (default 587) |
| `SMTP_USER` / `EMAIL_USER` | SMTP auth username |
| `SMTP_PASS` / `EMAIL_PASS` | SMTP auth password |
| `RESEND_API_KEY` | Resend.com API key (preferred over SMTP) |
| `EMAIL_FROM` | Sender address in outgoing verification emails |
| `GEMINI_API_KEY` | For AI-assisted course generation (unrelated to auth) |

---

*Derived from: SpatialCourse_Crafter — FunGIS Spatial Olympics Platform*  
*Pattern applies to any RBAC app with teams and project/course assignments backed by a lightweight JSON API.*
