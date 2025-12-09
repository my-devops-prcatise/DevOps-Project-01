
***

##  README.md

# How to Copy a Single Project from a Forked Repository into a New GitHub Repository

This guide explains how to extract **only one project folder** (e.g., `DevOps-Project-01`) from a large forked repository and push it to a **new GitHub repository**.

***

##  Prerequisites

*   **Git installed** on your system.
*   Access to the **original forked repository**.
*   A **new GitHub repository** created under your account or organization.

***

##  Steps to Copy and Push a Single Project

### **1. Navigate to the Project Folder**

Move into the directory you want to extract:

```bash
cd ~/DevOps-Projects-Beginner-to-Advanced/DevOps-Project-01
```

***

### **2. Initialize a New Git Repository**

Create a fresh Git repository inside this folder:

```bash
git init
```

***

### **3. Rename the Default Branch to `main`**

```bash
git branch -M main
```

***

### **4. Stage All Files**

```bash
git add .
```

***

### **5. Commit the Files**

```bash
git commit -m "Initial commit: DevOps-Project-01"
```

***

### **6. Create a New Repository on GitHub**

*   Go to your GitHub account or organization.
*   Click **New Repository**.
*   Name it: `DevOps-Project-01`.
*   Do **NOT** initialize with README or .gitignore (we already have them).

***

### **7. Add the Remote URL**

Replace `<your-org>` with your organization name:

```bash
git remote add origin https://github.com/my-devops-prcatise/DevOps-Project-01.git
```

***

### **8. Push to GitHub and Set Upstream**

```bash
git push -u origin main
```

***

##  Result

Your new GitHub repository now contains **only the selected project folder** with its files.

***


***

##  Next Steps

*   Add a **README.md** for the project itself.
*   Configure **CI/CD pipeline** (e.g., GitHub Actions).
*   Add **Dockerfile** if containerization is needed.

***

###  Example Repository Structure After Extraction:

    DevOps-Project-01/
    ├── Java-Login-App/
    │   ├── pom.xml
    │   ├── src/
    │   └── README.md
    ├── infrastructure/
    │   ├── main.tf
    │   ├── modules/
    │   └── variables.tf
    └── README.md

***


👉 Should I prepare that next?
