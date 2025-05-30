# BCS 4103 Advanced Database Systems - Project
## Optimizing Database Performance with PostgreSQL and Node.js for Wine Quality Dataset

**Group Name:** `A`
**GitHub Repository:** `[https://github.com/BSCNRB344123/ADVANCED-DB-GROUP-A-2025]`
**Live API Documentation (when running):** [http://localhost:3000/api-docs](http://localhost:3000/api-docs)

---

## 1. Project Goal

This project implements and optimizes a PostgreSQL database system for the UCI Wine Quality dataset. It demonstrates schema design (3NF), backend API development (Node.js/Express), interactive API documentation (Swagger), performance tuning via indexes, stored procedures, and triggers, and secure connectivity to a cloud-hosted database on Oracle Cloud Infrastructure (OCI).

---

## 2. Technology Stack

*   **Database:** PostgreSQL (v14.x deployed on OCI)
*   **Cloud Platform:** Oracle Cloud Infrastructure (OCI)
    *   OCI Database with PostgreSQL (Managed Service)
    *   OCI Compute (for Bastion Host VM)
    *   OCI Virtual Cloud Network (VCN), Subnets, Security Lists/NSGs, Internet Gateway
*   **Backend:** Node.js, Express.js
*   **Database Driver:** `node-postgres` (npm package: `pg`)
*   **API Documentation:** Swagger UI (`swagger-ui-express`), Swagger JSDoc (`swagger-jsdoc`)
*   **Data Handling:** `csv-parser`
*   **Environment Variables:** `dotenv`
*   **Connectivity:** SSH Client (OpenSSH, Git Bash, or PuTTY) for tunneling
*   **Version Control:** Git, GitHub

---

## 3. Dataset Used

*   **Name:** Wine Quality Dataset (Combined Red & White)
*   **Source:** UC Irvine Machine Learning Repository
*   **Link:** [https://archive.ics.uci.edu/ml/datasets/wine+quality](https://archive.ics.uci.edu/ml/datasets/wine+quality)
*   **Files Required:** `winequality-red.csv`, `winequality-white.csv` (Must be placed in the `data/` directory).

---

## 4. Prerequisites (Local Machine Setup)

Ensure the following software is installed on the machine where you intend to run the setup and application:

1.  **Node.js and npm:** LTS version recommended ([https://nodejs.org/](https://nodejs.org/)). Verify installation by running `node -v` and `npm -v` in your terminal.
2.  **Git:** Required for cloning the repository ([https://git-scm.com/](https://git-scm.com/)).
3.  **PostgreSQL Client (`psql`):** Required for running `.sql` setup scripts. It's included with a full PostgreSQL installation or can often be installed separately. Verify by running `psql --version`.
4.  **SSH Client:** **Essential** for creating the secure tunnel to the OCI Bastion host. Choose ONE of these options:
    *   **Windows:**
        *   **OpenSSH Client (Recommended):** Check if installed via Windows Settings -> Apps -> Optional features. If not, install it from there. Verify by opening Command Prompt or PowerShell and typing `ssh`.
        *   **Git Bash:** Included with Git for Windows installation. Provides a Linux-like environment with `ssh`.
        *   **PuTTY:** Download `putty.exe` and `puttygen.exe` from the official website ([https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)). Requires key conversion to `.ppk` format using `puttygen.exe`.
    *   **macOS/Linux:** The `ssh` command is typically built-in. Verify by opening a terminal and typing `ssh`.

---

## 5. Setup Instructions (Execute in Order)

**STEP 5.1: OCI Infrastructure Setup (Manual - OCI Console)**

*(These steps must be performed manually within your OCI account console before proceeding with local setup.)*

1.  **Provision OCI Database with PostgreSQL:**
    *   Create a DB System using the managed PostgreSQL service.
    *   Place it in a **Private Subnet** within your chosen VCN.
    *   During creation, set and **securely record the initial administrative password** (usually for the `postgres` user).
    *   Note the **Database Name** you intend to use (e.g., `wine_quality`).
    *   Note the **Private IP Endpoint** (e.g., `10.0.1.111`) shown in the connection details.
    *   Download the **CA Certificate Bundle** from the connection details. Based on your structure, you seem to have named this `CaCertificate-wine_quality.pub`. **Verify this is the CA bundle and not an SSH key.** CA bundles are usually `.pem` files.
2.  **Create Public Subnet (if needed):**
    *   Ensure your VCN has a public subnet (uses a Route Table with a rule pointing `0.0.0.0/0` to an Internet Gateway). Create one if necessary (e.g., `public-subnet-bastion`).
3.  **Create Bastion Host (Compute Instance):**
    *   Launch a small Compute Instance (e.g., Oracle Linux `VM.Standard.E2.1.Micro`) in the **Public Subnet**.
    *   **Crucially:** Ensure "Assign a public IPv4 address" is enabled during creation.
    *   Configure SSH keys: **Generate a new key pair** via OCI and **SAVE BOTH** the public and private key files securely, OR **upload your existing public key**. You MUST have the corresponding private key file locally.
    *   Note the Bastion's assigned **Public IP Address**, **Private IP Address**, and default **OS Username** (`opc` for Oracle Linux, `ubuntu` for Ubuntu).
4.  **Configure Network Security (Security Lists / NSGs):**
    *   **Rule 1 (SSH to Bastion):** In the Security List/NSG associated with the **Bastion's Public Subnet**, add an **Ingress Rule**:
        *   Source: `Your Local Machine's Public IP Address/32` (Find via "what is my IP address")
        *   Protocol: TCP
        *   Destination Port: `22`
    *   **Rule 2 (PG from Bastion to DB):** In the Security List/NSG associated with the **Database's Private Subnet** (e.g., `wine-quality-security-group`), add an **Ingress Rule**:
        *   Source: `Bastion's Private IP Address/32`
        *   Protocol: TCP
        *   Destination Port: `5432` (or your DB port)

**STEP 5.2: Local Code Setup**

1.  **Clone Repository:** Open your local terminal or Git Bash.
    ```bash
    git clone [https://github.com/BSCNRB344123/ADVANCED-DB-GROUP-A-2025]
    cd [repository-folder-name]
    ```
2.  **Place Data Files:** Ensure `winequality-red.csv` and `winequality-white.csv` are inside the `data/` directory within the cloned project folder.

**STEP 5.3: Establish SSH Tunnel**

*(This creates the secure connection path. Choose **EITHER** Command Line **OR** PuTTY.)*

*   **Open a dedicated Terminal/Command Prompt/Git Bash window** for the tunnel. This window **must remain open** while you work with the database/API.

*   **OPTION A: Command Line SSH (Recommended for Windows OpenSSH/Git Bash/macOS/Linux):**
    1.  Construct the command (replace *all* placeholders):
        ```bash
        ssh -N -L <local_port>:<db_private_ip>:<db_port> <bastion_user>@<bastion_public_ip> -i "<path_to_your_private_key>"
        ```
        *   `<local_port>`: **`5433`** (Recommended to avoid clashes)
        *   `<db_private_ip>`: The Private IP of your OCI Database (e.g., `10.0.1.111`)
        *   `<db_port>`: `5432`
        *   `<bastion_user>`: `opc` or `ubuntu` (Username for the Bastion VM OS)
        *   `<bastion_public_ip>`: Public IP address of your Bastion VM.
        *   `<path_to_your_private_key>`: **Exact, full path** to the SSH private key file you saved for the bastion. **Use "Copy as path" in Windows Explorer or provide the full path.** Ensure the path is correct and the file exists (fix `No such file or directory` errors). Use quotes around the path if it contains spaces.
    2.  Run the command. Accept the host key (`yes`) if prompted on first connection.
    3.  **Verification:** The command should connect successfully and then **appear to hang** without returning to the command prompt. This indicates the tunnel is active. Leave this window running. Troubleshoot `Connection timed out` errors by checking the Bastion Public IP and the OCI Security Rule allowing SSH (Port 22) from your *current* local public IP. Troubleshoot `Permission denied` errors by verifying the key path, username, and key permissions (if on Linux/macOS).

*   **OPTION B: PuTTY (Windows Graphical Client):**
    1.  **Convert Key:** If your private key isn't `.ppk`, use `puttygen.exe` -> Load -> Select key -> Save private key (optionally add passphrase) -> Save as `.ppk`.
    2.  **Run `putty.exe`**.
    3.  **Session:** Host Name=`<bastion_public_ip>`, Port=`22`, Connection type=`SSH`.
    4.  **Connection -> Data:** Auto-login username=`<bastion_user>`.
    5.  **Connection -> SSH -> Auth -> Credentials:** Browse and select your **`.ppk`** private key file.
    6.  **Connection -> SSH -> Tunnels:**
        *   Source port: **`5433`** (Recommended local port)
        *   Destination: `<db_private_ip>:<db_port>` (e.g., `10.0.1.111:5432`)
        *   Select `Local` and `Auto`.
        *   Click **`Add`**. Verify the rule appears in the list box.
    7.  **(Optional) Session:** Save the session details for future use.
    8.  Click **`Open`**. Accept the host key if prompted. Enter key passphrase if you set one.
    9.  **Verification:** A PuTTY terminal window should open and log you into the bastion host. **Keep this PuTTY window open** to maintain the tunnel.

**STEP 5.4: Database Initialization**

*   Open a **NEW** local terminal window (different from the tunnel window).
*   Ensure the SSH Tunnel from Step 5.3 is active.

1.  **Connect as Admin User:**
    ```bash
    psql -h localhost -p 5433 -U <Your_OCI_DB_Admin_User> -d postgres
    ```
    *   Replace `<Your_OCI_DB_Admin_User>` (e.g., `postgres`).
    *   Enter the OCI DB Admin password set during DB System creation when prompted.

2.  **Create Application Database (Inside `psql`):**
    ```sql
    CREATE DATABASE wine_quality;
    ```
    *   Verify success (`CREATE DATABASE` message). Check if it already exists first with `\l` if needed.

3.  **Exit `psql`:**
    ```sql
    \q
    ```

4.  **Apply Schema:**
    ```bash
    psql -h localhost -p 5433 -U <Your_OCI_DB_Admin_User> -d wine_quality -f database/schema.sql
    ```
    *   Enter admin password again. Verify no errors.

5.  **Apply Optimizations:**
    ```bash
    psql -h localhost -p 5433 -U <Your_OCI_DB_Admin_User> -d wine_quality -f database/optimizations.sql
    ```
    *   Enter admin password again. Verify no errors.

6.  **(Optional but Recommended) Create App User & Grant Privileges:**
    *   Connect again as admin: `psql -h localhost -p 5433 -U <Your_OCI_DB_Admin_User> -d wine_quality`
    *   Inside `psql`, run:
        ```sql
        -- Use a strong password for your app user!
        CREATE USER donnelly WITH PASSWORD 'donN1234#'; -- Example user/password from history
        GRANT CONNECT ON DATABASE wine_quality TO donnelly;
        GRANT USAGE ON SCHEMA public TO donnelly;
        GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE wines, wine_audit_log TO donnelly;
        GRANT USAGE, SELECT ON SEQUENCE wines_id_seq, wine_audit_log_log_id_seq TO donnelly;
        -- Add other grants if needed (e.g., EXECUTE on functions)
        ```
    *   Exit `psql`: `\q`

**STEP 5.5: Backend API Configuration**

1.  **Navigate to Backend Folder:**
    ```bash
    cd wine-quality-backend # Navigate to your backend application folder
    ```

2.  **Edit `.env` File:** Open the `.env` file and carefully populate it. **Use the dedicated app user/password (e.g., `donnelly`) if you created one.**
    ```env
    # === EDIT THESE VALUES CAREFULLY ===
    DB_USER=USERNAME                     # The user the Node.js app connects as
    DB_PASSWORD=your_password                 # The password for DB_USER (NO QUOTES!)
    DB_DATABASE=DB_NAME              # The database created in Step 5.4
    DB_HOST=localhost                     # Connect via the tunnel
    DB_PORT=5433                          # The LOCAL port used in the tunnel command

    # SSL Settings - Required for OCI
    DB_SSL_ENABLED=true
    DB_SSL_REJECT_UNAUTHORIZED=false      # NEEDED for tunnel connection to localhost
    DB_SSL_CA_CERT_FILENAME=CaCertificate-wine_quality.pub # MATCHES YOUR STRUCTURE - VERIFY IT'S A CA BUNDLE!

    # Server Port
    PORT=3000
    # === END EDIT ===
    ```
    *   **CRITICAL:** Ensure `DB_PASSWORD` has no quotes around it.
    *   **CRITICAL:** Ensure `DB_SSL_CA_CERT_FILENAME` matches the exact name of the file in your `certs/` folder. **Verify this file is the CA Bundle (usually `.pem`) and not an SSH key (`.pub`).**
    *   **CRITICAL:** Place the downloaded CA certificate file (e.g., `CaCertificate-wine_quality.pub` based on your structure) inside the `wine-quality-backend/certs/` directory.

3.  **Install Dependencies:** While still in the `wine-quality-backend` directory:
    ```bash
    npm install
    ```

**STEP 5.6: Populate Database**

1.  **Navigate to Project Root:**
    ```bash
    cd ..
    ```
2.  **Ensure SSH Tunnel is Active.**
3.  **Run Population Script:**
    ```bash
    node database/populate.js
    ```
    *   Watch the console for success messages or errors (e.g., authentication failed, database doesn't exist - troubleshoot based on previous steps if errors occur).

## 6. Running the Application

1.  **Ensure SSH Tunnel is Active** (Terminal window from Step 5.3 is still open and connected/hanging).
2.  Open a **NEW** terminal window.
3.  Navigate to the backend directory:
    ```bash
    cd wine-quality-backend
    ```
4.  Start the Node.js server:
    ```bash
    npm start
    ```
5.  **Verification:** Look for console output indicating the server is running and database connection pool is created without errors (especially check for password authentication or SSL errors). You should see output similar to:
    ```
    Server running on port 3000
    Attempting to connect to DB: wine_quality on localhost:5433 as donnelly
    Attempting to load CA certificate from: ...\certs\CaCertificate-wine_quality.pub
    SSL Configuration Enabled. rejectUnauthorized: false
    Database connection pool created. Client connected. SSL: Enabled
    Swagger docs available at http://localhost:3000/api-docs
    ```

## 7. Accessing the API & Documentation

*   **Swagger UI:** Open a web browser and navigate to `http://localhost:3000/api-docs`.
*   Use the UI to explore endpoints and click "Try it out" -> "Execute" to test API calls.
*   **Verify Connection:** Successfully executing `GET /api/wines` and receiving data confirms the connection to the OCI database is working through the tunnel.

## 8. Key Features & Optimizations

*   **3NF Schema:** Database normalized to reduce redundancy (removed calculated `is_premium` column).
*   **Indexing:** Indexes on `quality`, `alcohol`, `wine_type` for faster querying.
*   **Stored Procedure:** `calculate_average_quality` for efficient server-side calculation.
*   **Triggers:** Automated `updated_at` timestamp management and audit logging for `quality` changes.
*   **Secure Cloud Connectivity:** Use of OCI private database endpoint accessed via SSH Tunneling through a Bastion Host.

## 9. Project Structure

```plaintext
[repository-folder-name]/
├── .gitignore
├── data/                 # CSV data files must be placed here
│   ├── winequality-red.csv
│   └── winequality-white.csv
├── database/             # Database scripts and tools
│   ├── optimizations.sql # Stored Procedures & Triggers
│   ├── populate.js       # Data population script
│   └── schema.sql        # 3NF Database schema SQL
├── wine-quality-backend/ # Node.js application
│   ├── certs/            # OCI CA certificate bundle must be placed here
│   │   └── CaCertificate-wine_quality.pub # Filename from your structure
│   ├── config/           # Configuration files
│   │   ├── db.config.js  # Database connection pool setup
│   │   └── swagger.js    # Swagger setup
│   ├── controllers/      # Request handling logic
│   │   └── wineController.js
│   ├── node_modules/     # NPM packages (generated)
│   ├── routes/           # API endpoint definitions
│   │   └── wine.routes.js
│   ├── .env              # Local environment config (!!! DO NOT COMMIT !!!)
│   ├── package-lock.json
│   ├── package.json
│   └── server.js         # Express server entry point
├── README.md             # This file
└── REPORT.md             # Detailed project report document
```

## 10. Troubleshooting Quick Reference

*   **`ssh` not recognized:** Install OpenSSH Client (Windows Optional Features) or use Git Bash / PuTTY. Check PATH.
*   **SSH Key `No such file or directory`:** Verify the `-i` path is exact. Use "Copy as path". Check filename.
*   **SSH `Connection timed out`:** Verify Bastion Public IP. Check OCI Security Rule for Port 22 allows *your current* public IP.
*   **SSH `Permission denied (publickey)`:** Verify `-i` key path. Check key permissions (Linux/macOS: `chmod 400`). Ensure correct key pair used. Verify bastion username (`opc`/`ubuntu`).
*   **PuTTY Key:** Use `puttygen.exe` to convert key to `.ppk` format if needed.
*   **PuTTY Tunnel Not Working:** Ensure tunnel rule (`L5433 ...`) was **Added** in the Tunnels config. Keep PuTTY window open.
*   **Node `ETIMEDOUT` / `ECONNREFUSED`:** SSH Tunnel not running or not listening on the correct local port (`5433`). Check OCI Security Rule for Port `5432` allows Bastion Private IP -> DB Private Subnet/NSG.
*   **Node `ENOTFOUND`:** `DB_HOST` in `.env` is wrong. Should be `localhost` when using tunnel.
*   **Node `password authentication failed for user "..."`:** Incorrect `DB_PASSWORD` or `DB_USER` in `.env`. Reset password in `psql` if unsure. Remove quotes around password in `.env`.
*   **Node `database ... does not exist`:** Database specified in `DB_DATABASE` doesn't exist on OCI server. Run `CREATE DATABASE ...;` using `psql` as admin.
*   **Node SSL `Hostname/IP does not match...`:** Set `DB_SSL_REJECT_UNAUTHORIZED=false` in `.env`.
*   **Node SSL `certificate verify failed` / `unknown CA`:** Verify `DB_SSL_CA_CERT_FILENAME` in `.env` matches the actual CA bundle file name (check if `.pub` is correct, usually `.pem`). Ensure file is in `wine-quality-backend/certs/`. Download the correct CA bundle from OCI.
*   **YAML Errors on `npm start`:** Check JSDoc comments in `routes/wine.routes.js` for incorrect indentation or use of tabs instead of spaces. Replace the main `components:` block if needed.

## 11. Group Members

*   `Ian Mwaniki`
*   `Edgar Nyolo`
*   `Donnelly Amaitsa`
*   `Gloria Abineza`
*   `Safia Jamal`

---
