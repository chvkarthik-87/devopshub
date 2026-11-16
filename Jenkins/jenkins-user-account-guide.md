### 🔹 How Users Can Create Their Own Accounts in Jenkins (Self-Signup)

By default, Jenkins **disables self-registration** for security reasons. To allow users to create their own accounts:

1. **Edit Jenkins Configuration File** (as root or jenkins user):

   ```bash
   sudo vi /var/lib/jenkins/config.xml
   ```

2. **Find the `<disableSignup>` line** (usually near the top):

   ```xml
   <disableSignup>true</disableSignup>
   ```

3. **Change it to `false`**:

   ```xml
   <disableSignup>false</disableSignup>
   ```

4. **Save and exit** the editor.

5. **Restart Jenkins** to apply changes:

   ```bash
   sudo systemctl restart jenkins
   ```

6. **Result**:
   - Now, on the login page, users will see a **"Sign up"** link.
   - Anyone can create their own account by providing username, password, full name, and email.

**Security Note**: Enabling self-signup is **not recommended** for production or internet-facing Jenkins instances (risk of spam/abuse). Use only in trusted internal environments.

### 🔹 How to Restrict Users from Full Access (Role-Based or Matrix Authorization)

To prevent users from having full admin rights and give limited permissions:

1. Log in as an **admin user**.

2. Go to **Manage Jenkins → Security** (or **Configure Global Security** in older versions).

3. Under **Authorization**, select one of these strategies:

   - **Recommended**: **Project-based Matrix Authorization Strategy**  
     (Allows fine-grained control per job/folder)

   - **Alternative**: **Role-Based Strategy** (requires installing the "Role-based Authorization Strategy" plugin)

4. **Using Project-based Matrix Authorization Strategy**:

   - Scroll down to the matrix table.
   - Add users or groups in the left column (e.g., `developer1`, `authenticated` group for all logged-in users).
   - Grant only necessary permissions (check boxes):

     | Permission             | Recommended for Limited Users |
     |------------------------|-------------------------------|
     | Overall → Read         | ✅ (to view dashboard)        |
     | Job → Read             | ✅                            |
     | Job → Build            | ✅ (to trigger builds)        |
     | Job → Workspace        | ✅ (view files)               |
     | Job → Cancel           | Optional                     |
     | Job → Create/Delete    | ❌ (admin only)               |
     | Job → Configure        | ❌                            |
     | Overall → Administer   | ❌                            |

   - **Do NOT check** dangerous permissions like:
     - Overall → Administer
     - Agent → Configure / Disconnect
     - Run → Scripts

5. Click **Save**.

**Result**:
- Restricted users can:
  - Log in
  - View and trigger assigned jobs
  - See build history and logs
- They **cannot**:
  - Create/delete jobs
  - Configure Jenkins globally
  - Manage agents or security

**Best Practices**:
- Use **groups** (e.g., `developers`, `viewers`) instead of individual users.
- Install **Role-based Authorization Strategy** plugin for folder/project-level roles.
- Enable **Audit Trail** plugin to log who did what.

**References**:
- https://www.jenkins.io/doc/book/security/access-control/
