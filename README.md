# JENKinsnotes.md-

<img width="1223" height="369" alt="image" src="https://github.com/user-attachments/assets/44b7a2f6-82e4-40dc-b0d3-7aa14b0fc487" />

## This warning is serious but very common in Jenkins setups — especially when you build Docker/Angular projects.

## I’ll explain why it happens and give you exact, safe steps to fix it now and prevent it permanently.

## IMMEDIATE FIX (DO THIS NOW)

1️⃣ CLEAN OLD WORKSPACES

💡 Jenkins will recreate workspaces automatically.
````
sudo systemctl stop jenkins
sudo rm -rf /var/lib/jenkins/workspace/*
sudo systemctl start jenkins
````
🔥 2️⃣ DELETE OLD BUILDS (VERY IMPORTANT)

This removes old build history (logs/artifacts).
````
sudo rm -rf /var/lib/jenkins/jobs/*/builds/*
````
🔥 3️⃣ CLEAN DOCKER (MAJOR SPACE SAVER)
````
docker system prune -af
docker builder prune -af
docker volume prune -f
````

🔥 4️⃣ CHECK SPACE
````
df -h
du -sh /var/lib/jenkins/*
````

# ✅ PERMANENT PREVENTION (RECOMMENDED)

🛡️ 1️⃣ ENABLE BUILD ROTATION (Jenkins UI)

For Each JOb:

````
Job → Configure → Discard Old Builds
✔ Keep only last 5–10 builds
✔ Days to keep builds: 7
````

🛡️ 2️⃣ DELETE WORKSPACE AFTER BUILD (BEST PRACTICE)

Jenkinfile

````
post {
  always {
    cleanWs()
  }
}
````
This automatically cleans workspace after every build.

🛡️ 3️⃣ MOVE JENKINS_HOME TO BIGGER DISK (BEST LONG-TERM)

Example (new disk at /data):

````
sudo systemctl stop jenkins
sudo mv /var/lib/jenkins /data/jenkins
sudo ln -s /data/jenkins /var/lib/jenkins
sudo systemctl start jenkins
````

🛡️ 4️⃣ EXCLUDE HEAVY FILES FROM DOCKER BUILDS

Create .dockerignore (you already learned this):

````
node_modules
.angular
dist
.git
````

# but iam not even getting access to ssh the jenkins server for that what to do -- for that we can do is 

✅ OPTION 1: CLEAN SPACE DIRECTLY FROM JENKINS UI (NO SSH)

🟢 1️⃣ Delete Old Builds via UI (FAST)

Open Jenkins → Job

Click Build History

Delete old builds manually (❌ button)

✔ This frees /var/lib/jenkins/jobs/*/builds/

🟢 2️⃣ Configure “Discard Old Builds” (VERY IMPORTANT)

For EACH job:

````
Job → Configure
✔ Discard old builds
Max builds: 5
Max days: 7
Save
````
✔ Prevents future disk issues

🟢 3️⃣ Delete Workspace via UI

Jenkins → Job

Click Workspace

Click Wipe Out Workspace

✔ Frees /var/lib/jenkins/workspace

🟢 4️⃣ Remove Artifacts (If configured)

Go to build

Click Artifacts

Delete heavy zip/dist files

✅ OPTION 2: USE SCRIPT CONSOLE (NO SSH – POWERFUL)

✅ OPTION 3: IF JENKINS IS ON CLOUD (AWS/GCP/Azure)

Even without SSH, you can:

🔹 Increase Disk Size (NO DATA LOSS)
AWS EC2:

EC2 → Volumes → Modify Volume → Increase size

GCP Compute Engine:

Edit disk → Increase size → Reboot VM

Azure:

Resize Disk → Restart VM

✔ Jenkins automatically gets more space


