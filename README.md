# Enterprise Inventory API & Data Pipeline Utility

An automated API-to-database integration and governance utility built with **PostgreSQL (Supabase)** and **Postman collection automation**. This project demonstrates dynamic JWT token lifecycle management, programmatic response sanitization, and automated request workflows designed to enforce database security and streamline backend ingestion.

---

## Architecture & Core Features

* **Dynamic Auth & Token Lifecycle Management:** Uses Postman pre-request scripting to evaluate token expiration thresholds dynamically, automatically refreshing Supabase JWTs on-the-fly to prevent session drops during long test runs.
* **Automated Data Ingestion & Routing:** Connects directly to Supabase REST endpoints (`/rest/v1/`) to handle secure payload delivery and transaction verification.
* **Programmatic Response Sanitization & Masking:** Includes custom JavaScript test scripts that recursively parse incoming JSON responses, identifying sensitive identifiers (such as `user_id`, `profiles_id`, and `id`) and masking them before logging outputs to the console.
* **Database Security Alignment:** Designed to interface seamlessly with PostgreSQL Row-Level Security (RLS) policies, ensuring data ingestion strictly respects user-scoped access rules.

---
## Challenges Faced & Solutions

* **Resolving `401 Unauthorized` & Status Code Mismatches:** Early batch runs threw unexpected authentication errors and empty streams when interacting with endpoints. This was resolved by implementing proactive pre-request threshold checks (refreshing tokens within a 300-second window) and ensuring headers explicitly requested response bodies (`Prefer: return=representation`).

* **Enforcing Granular RLS & Role Permissions:** Securing tables against public exposure while allowing authenticated API workflows required moving past default configurations. By explicitly mapping schema-level grants to the `authenticated` role and aligning Row-Level Security policies with `auth.uid()`, data integrity and user-scoped isolation were successfully enforced without blocking valid CRUD operations.
## Project Structure

```text
├── enterprise-inventory-api.postman_collection.json    # Main Postman collection with pre-request auth & test-script sanitization
└── README.md                                        # Project documentation
```

## Getting Started

### Prerequisites
* Postman (Desktop App or Web)
* A backend target (such as a Supabase project or a mock server) configured with matching API routes and database tables.

### Environment Variables Setup
To run this collection successfully, configure the following variables in your Postman Environment:

| Variable | Description |
| :--- | :--- |
| `supa_url` | Your target API base URL |
| `supa_anon_key` | Your public API key or anon token |
| `user_email` | Test user email for automated auth refreshing |
| `user_password` | Test user password |
| `jwt_token` | Automatically populated/refreshed by the pre-request script |
| `token_expiry` | Epoch timestamp tracking active token expiration |

### Import & Execution
* Download or clone this repository.
* Open Postman and click Import, then select `enterprise-inventory-api.postman_collection.json`.
* Link your active Postman Environment containing the variables listed above.
* Run requests individually or execute the collection via the Postman Collection Runner to trigger automated token refreshing and response sanitization.

### Technical Highlights
The pre-request script handles authentication validation cleanly before any network call is dispatched:

```javascript
if (!userJwt || (tokenExpiry - currentTime) < 300) {
    console.log("Token expired or missing. Fetching new token...");
    pm.sendRequest({
        url: supaUrl + "/auth/v1/token?grant_type=password",
        method: 'POST',
        header: { 'Content-Type': 'application/json', 'apikey': supaKey },
        body: { mode: 'raw', raw: JSON.stringify({ email: userEmail, password: userPassword }) }
    }, function (err, res) {
        // Sets new JWT environment variables upon successful auth
    });
}
```

### Troubleshooting & Edge Cases

If you run into issues getting Supabase REST endpoints to return proper status codes with empty or minimal payloads, ensure your headers include the correct `Prefer` representation preference:

```json
// Ensure headers match Postman collection configuration
"header": [
    { "key": "apikey", "value": "{{supa_anon_key}}" },
    { "key": "Content-Type", "value": "application/json" },
    { "key": "Prefer", "value": "return=representation" }
]
```

### Troubleshooting, Diagnostics & Schema Setup

If you run into permission drop-offs, unexpected 401 Unauthorized responses, or status code mismatches during automated test runs, use the diagnostic and setup scripts below. (Replace your_table_name with your actual target table).
1. Diagnostic & Permission Verification Queries

Run these queries to inspect role privileges and ensure Row-Level Security is properly active:

```SQL
-- Check table privileges for the authenticated role
SELECT grantee, privilege_type 
FROM information_schema.role_table_grants 
WHERE table_name = 'your_table_name';

-- Verify current Row-Level Security status on public tables
SELECT tablename, rowsecurity 
FROM pg_tables 
WHERE schemaname = 'public';
```

 ### Granting Schema & Role Access

Ensure API roles have proper usage and CRUD grants to prevent requests from being blocked at the schema level:

```SQL
-- Grant usage on the public schema to API roles
GRANT USAGE ON SCHEMA public TO anon, authenticated;

-- Grant explicit table-level permissions
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO authenticated;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO authenticated;

```


### Complete Table Schema Reset & RLS Configuration

If your target table needs a clean slate to enforce clean 200 OK transaction handling and user scoping, run this full teardown and rebuild script:

```SQL
-- 1. Drop existing policies and tables to clear out corrupted or conflicting state
DROP POLICY IF EXISTS "Enable user-scoped CRUD access" ON public.your_table_name;
DROP TABLE IF EXISTS public.your_table_name;

-- 2. Recreate the table with proper foreign key linking to Supabase auth users
CREATE TABLE public.your_table_name (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
    item_name TEXT NOT NULL,
    quantity INTEGER NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- 3. Re-enable Row-Level Security
ALTER TABLE public.your_table_name ENABLE ROW LEVEL SECURITY;

-- 4. Re-apply strict user-scoped security policy
CREATE POLICY "Enable user-scoped CRUD access" ON public.your_table_name
    FOR ALL
    TO authenticated
    USING (auth.uid() = user_id)
    WITH CHECK (auth.uid() = user_id);

-- 5. Ensure schema-level and role-level grants are properly aligned
GRANT USAGE ON SCHEMA public TO anon, authenticated;
GRANT SELECT, INSERT, UPDATE, DELETE ON public.your_table_name TO authenticated;

```




### Q: Why use Content-Type: application/json instead of form-urlencoded or raw text?

Answer: Our payload requires structured data types—like nested objects and booleans—that raw text or form formats can't preserve. It establishes a strict contract via HTTP headers so the server's body parser knows how to deserialize incoming bytes before handling authentication.

### Q: What is the purpose of setting body: { mode: 'raw', raw: JSON.stringify(...) }?

    
Answer: Network protocols require text or byte streams. JSON.stringify() serializes our JavaScript object into a flat string, and mode: 'raw' forces Postman to deliver that exact string into the request body without altering, escaping, or form-encoding it.

### Q: Why implement a 5-minute pre-request buffer for JWT expiration?

Answer: Checking (tokenExpiry - currentTime) < 300 proactively refreshes the token before it officially dies, preventing race conditions or mid-test authentication failures during automated collection runs.
