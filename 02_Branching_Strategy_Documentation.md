# DevOps Environment & Branching Strategy

This documentation provides a **detailed explanation** of the concepts such as DevOps environments, branching strategies, GitHub organization setup, RBAC (Role-Based Access Control), and branch protection rules.  

---

## 1. DevOps Environments  

In DevOps, we typically work with **five key environments**, each serving a specific purpose in the software development lifecycle.  

### 1.1 Development (DEV) Environment  
- **Purpose:** Testing and running **newly developed features**.  
- Developers write new code (e.g., adding a button, changing background color).  
- This code is first deployed in the DEV environment (often a Kubernetes cluster).  
- The DEV environment is also known as the **lower/lowest environment**.  

### 1.2 Quality Assurance (QA) Environment  
- **Purpose:** Detecting **bugs and issues** in the code.  
- QA engineers/testers use tools like Selenium, Cypress, or Robot Framework to run automated/manual tests.  
- The focus is on **bug detection and fixing** before moving to higher environments.  

### 1.3 Pre-Production (PPD) Environment  
- **Purpose:** Simulates the **production environment** before actual release.  
- Same configuration as PROD to test real-world behavior.  
- Ensures stability before deployment to PROD.  

### 1.4 Production (PROD) Environment  
- **Purpose:** The **live environment** where applications are accessible to users.  
- For public applications (e.g., Zomato), this is accessed by the general public.  
- For internal apps, only authorized users/clients access it.  
- Must be **bug-free, stable, and secure**.  

### 1.5 Disaster Recovery (DR) Environment  
- **Purpose:** Backup environment to take over when PROD crashes.  
- Same configuration as PROD.  
- Users are redirected to DR if PROD goes down.  
- Used only during emergencies to ensure **high availability**.  

---

## 2. Branching Strategies  

Branching strategies determine how code flows from **feature development → testing → production**.  

### 2.1 General Branching Strategy  
Each environment has a **dedicated branch**:  
- `dev` → Development  
- `qa` → Quality Assurance  
- `ppd` → Pre-Production  
- `main` → Production  
- `dr` → Disaster Recovery  

**Workflow:**  
1. Developers create **feature branches** (e.g., `f_sub_1/bg-change`, `f_sub_2/banner`).  
2. Code is tested in `dev`.  
3. Merged into `qa` → bugs fixed in **bug-fix branches**.  
4. Merged into `ppd` → tested in a PROD-like setup.  
5. Merged into `main` → deployed to production.  
6. If issues arise in PROD → **hotfix branch** is created immediately.  
7. Final code synced with `dr` branch for backup.  

### 2.2 Cost-Optimized Branching Strategy (Used in Companies)  
- Instead of **5 clusters**, companies often use **2 clusters**:  
  - **Cluster 1:** DEV + QA + PPD  
  - **Cluster 2:** PROD  
- Branches: `dev` and `main`.  
- Testing and pre-production happen within the same cluster → **cost-efficient**.  
- DR may not be maintained due to cost. Instead, **hotfixes** are applied quickly.  

**Release Candidate (RC) Flow:**  
- New version (v2) → `RC1`, `RC2`, `RC3` created from `dev`.  
- Iterative bug fixes until stable.  
- Final merge to `main` → deploy to PROD.  

---

## 3. GitHub Organization Setup  

### 3.1 Why Organizations?  
- Companies prefer creating repositories inside **GitHub Organizations** for better management.  
- Provides centralized **access control** and **team management**.  

### 3.2 Teams and Roles (RBAC)  
Create teams inside the organization:  
1. **Developers** → Write access (push, commit, create branches).  
2. **Reviewers** → Triage access (read + manage pull requests).  
3. **DevOps** → Admin access (full control of repositories).  
4. **Readers/Auditors** → Read-only access (for compliance, audits).  

### 3.3 Base Permissions  
- Default access for members not part of any team.  
- Usually set to **Read** only.  

### 3.4 Repository-Level Permissions  
- Teams can be assigned to **specific repositories**.  
- Example: Developers may only have access to a single project, not all repos.  

---

## 4. Branch Protection Rules  

Branch protection ensures **stability and security** of critical branches.  

### 4.1 Why Branch Protection?  
- Prevent accidental pushes to `main` or release branches.  
- Ensure only **reviewed and approved PRs** get merged.  
- Prevent branch deletion or unauthorized modifications.  

### 4.2 Common Protection Rules  
1. **Require pull requests (PRs)** for merging into main/release branches.  
2. **Require at least 1 approval** before merging.  
3. **Restrict branch deletion** to avoid accidental loss.  
4. **Conversation resolution required** before PR merges.  
5. **Require recent review** to prevent unapproved changes.  
6. **Lock Branch** – No pushes allowed (used when project is archived).  

### 4.3 Example: Protecting `main` Branch  
- Restrict direct pushes.  
- Require PRs with minimum 1 approval.  
- Prevent deletion of `main`.  
- Allow only **admins** to bypass rules.  

---

## 5. Key Takeaways  
- **5 Environments**: DEV → QA → PPD → PROD → DR.  
- **Branching**: Feature → Dev → QA → PPD → Main → DR.  
- **Cost Saving**: Use only 2 clusters (Dev+QA+PPD, Prod).  
- **GitHub Org Setup**: Manage access with **teams & RBAC**.  
- **Branch Protection**: Secure critical branches with rules.  

---
  
