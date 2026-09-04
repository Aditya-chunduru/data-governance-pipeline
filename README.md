# Enterprise Inventory API & Data Pipeline Utility

An automated API testing and data governance utility built with PostgreSQL (Supabase), Postman collection automation, and GitHub Actions CI/CD pipelines. This project demonstrates sequential request orchestration, programmatic response sanitization, and automated contract regression testing designed to streamline backend catalog ingestion and maintain build log hygiene.

## Architecture & Core Features

* **Sequential Token Lifecycle Management:** Implements an ordered collection flow where an initial authentication request captures and caches Supabase JWTs into environment variables for subsequent protected endpoint execution.
* **Automated Data Ingestion & Routing:** Connects directly to Supabase REST endpoints (`/rest/v1/`) to handle secure payload delivery and transaction verification.
* **Programmatic Response Sanitization & Masking:** Includes custom JavaScript test scripts that recursively parse incoming JSON responses, identifying sensitive identifiers (such as `user_id`, `profiles_id`, and `id`) and masking them before logging outputs to the console.
* **CI/CD Quality Gate:** Fully integrated with GitHub Actions and Newman to execute automated test suites on every push, ensuring schema consistency and response reliability.

## Project Structure

```text
├── enterprise-inventory-api.postman_collection.json    # Main Postman collection with sequential auth & test-script sanitization
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
| `user_email` | Test user email for authentication |
| `user_password` | Test user password |
| `jwt_token` | Automatically populated/refreshed by the Authentication script |
| `token_expiry` | Epoch timestamp tracking active token expiration |

### Import & Execution
* Download or clone this repository.
* Open Postman and click Import, then select `enterprise-inventory-api.postman_collection.json`.
* Link your active Postman Environment containing the variables listed above.
* Run requests individually or execute the collection via the Postman Collection Runner to trigger token caching and response sanitization.

### Technical Highlights
The authentication test script captures the access token in its Test tab immediately upon a successful 200 OK response:

```javascript
// Wrap parsing in a try block to prevent CI/CD runner crashes if the server returns non-JSON or gateway error pages
try {
// Convert the raw HTTP response body into a structured JavaScript object
    const responseJson = pm.response.json();
    
 // Validate response code and verify token existence before caching
    if (pm.response.code === 200 && responseJson.access_token) {
        pm.environment.set("jwt_token", responseJson.access_token);
        
        // Calculate absolute token expiration timestamp if provided
        if (responseJson.expires_in) {
            const currentTime = Math.floor(Date.now() / 1000);
            pm.environment.set("token_expiry", currentTime + responseJson.expires_in);
        }
        
        console.log("Auth success: JWT token and expiration cached.");
    } else {
        console.error("Auth failed: Response missing access token or invalid status code.");
    }

 // Assert successful HTTP status
    pm.test("Auth status is 200", () => {
        pm.response.to.have.status(200);
    });

} catch (e) {
    console.error("Could not parse auth response as JSON.");
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

### Q: Why use Content-Type: application/json instead of form-urlencoded or raw text?

Answer: Our payload requires structured data types—like nested objects and booleans—that raw text or form formats can't preserve. It establishes a strict contract via HTTP headers so the server's body parser knows how to deserialize incoming bytes before handling authentication.

### Q: What is the purpose of setting body: { mode: 'raw', raw: JSON.stringify(...) }?

    
Answer: Network protocols require text or byte streams. JSON.stringify() serializes our JavaScript object into a flat string, and mode: 'raw' forces Postman to deliver that exact string into the request body without altering, escaping, or form-encoding it.

### Q: Why enforce a sequential request order in the collection?

A: Running the authentication endpoint first guarantees that a valid jwt_token is present in the environment variables before any protected business logic or database queries execute, preventing race conditions during automated CI runs.
