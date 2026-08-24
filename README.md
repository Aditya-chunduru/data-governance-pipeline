# Enterprise Inventory API & Data Pipeline Utility

An automated API-to-database integration and governance utility built with **PostgreSQL (Supabase)** and **Postman collection automation**. This project demonstrates dynamic JWT token lifecycle management, programmatic response sanitization, and automated request workflows designed to enforce database security and streamline backend ingestion.

---

## Architecture & Core Features

* **Dynamic Auth & Token Lifecycle Management:** Uses Postman pre-request scripting to evaluate token expiration thresholds dynamically, automatically refreshing Supabase JWTs on-the-fly to prevent session drops during long test runs.
* **Automated Data Ingestion & Routing:** Connects directly to Supabase REST endpoints (`/rest/v1/`) to handle secure payload delivery and transaction verification.
* **Programmatic Response Sanitization & Masking:** Includes custom JavaScript test scripts that recursively parse incoming JSON responses, identifying sensitive identifiers (such as `user_id`, `profiles_id`, and `id`) and masking them before logging outputs to the console.
* **Database Security Alignment:** Designed to interface seamlessly with PostgreSQL Row-Level Security (RLS) policies, ensuring data ingestion strictly respects user-scoped access rules.

---

## Project Structure

```text
├── enterprise-inventory-api.postman_collection.json    # Main Postman collection with pre-request auth & test-script sanitization
└── README.md                                           # Project documentation


## Getting Started

### Prerequisites
* **[Postman](https://www.postman.com/) (Desktop App or Web)
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
1. Download or clone this repository.
2. Open Postman and click **Import**, then select `enterprise-inventory-api.postman_collection.json`.
3. Link your active Postman Environment containing the variables listed above.
4. Run requests individually or execute the collection via the Postman Collection Runner to trigger automated token refreshing and response sanitization.

---

## Technical Highlights

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
