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
