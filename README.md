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

================================================================================

:: IF THE SPACE IS FULL YOU HAVE TO DO ONE-THING ::

# Confirm Disk Usage 

````
df -Th 
````

# Find what is eating space 

````
sudo du -xh / | sort -h | tail -20
````

# clean space 
````
sudo apt clean
sudo apt autoclean
sudo apt autoremove -y
````

============================================================================================
# Attached new disk to the vm --

OPTION A (RECOMMENDED): Attach a NEW data disk to the VM

Best for Jenkins / Docker / data storage

🔹 STEP 1: Create a new Persistent Disk

Go to GCP Console

Navigate to
Compute Engine → Disks

Click CREATE DISK

Fill details:

Name: jenkins-data-disk

Type: Balanced persistent disk

Size: 50 GB (or more)

Zone: us-central1-c ⚠️ (same as VM)

Encryption: Default

👉 Click Create

🔹 STEP 2: Attach disk to the VM

Go to
Compute Engine → VM instances

Click jenkins-server

Click Edit

Scroll to Additional disks

Click Attach existing disk

Select jenkins-data-disk

Mode: Read/Write

Click Save

✅ Disk is now attached to the VM

✅ STEP 3: Prepare the disk inside Linux (VERY IMPORTANT)

SSH into your VM.

3.1 Verify disk is visible
lsblk


You will see something like:

sdb    50G

3.2 Create filesystem
sudo mkfs.ext4 /dev/sdb

3.3 Create mount point
sudo mkdir /data

3.4 Mount disk
sudo mount /dev/sdb /data


Verify:

df -h


You should see:

/dev/sdb   50G   ...   /data

3.5 Make mount permanent (CRITICAL)
sudo blkid /dev/sdb


Copy the UUID.

Edit fstab:

sudo nano /etc/fstab


Add:

UUID=<UUID>   /data   ext4   defaults,nofail   0   2


Save & exit.

Test:

sudo mount -a

✅ STEP 4: Use disk for Jenkins (BEST PRACTICE)
Move Jenkins data

````
sudo systemctl stop jenkins
sudo rsync -avx /var/lib/jenkins/ /data/jenkins/
sudo mv /var/lib/jenkins /var/lib/jenkins_old
sudo ln -s /data/jenkins /var/lib/jenkins
sudo chown -R jenkins:jenkins /data/jenkins
````

Start Jenkins:

````
sudo systemctl start jenkins
sudo systemctl status jenkins
````

🔥 OPTION B (NOT RECOMMENDED)

Attaching volume as boot disk or resizing root partition — risky and unnecessary.

🧠 WHY THIS IS THE RIGHT WAY
Benefit	Reason
No OS break	Root disk untouched
Easy expansion	Resize data disk anytime
Stable Jenkins	No disk-full crashes
Industry standard	Used in prod setups
✅ Final verification

Run:

df -h
lsblk
sudo systemctl status jenkins
-----------------------------------------------

:::   ERROR to find this  :::

Credentials from Kubernetes Secrets will not be available.See the log for more details.

he key line is:

# SunCertPathBuilderException: unable to find valid certification path to requested target SSLHandshakeException: PKIX path building failed

That means:
👉 Jenkins does NOT trust the TLS certificate of your Kubernetes API server.

# Fix Option 1 (Best & Clean): Import Kubernetes CA into Jenkins Java Truststore

Step 1: Get Kubernetes cluster CA certificate

Run from Jenkins server (or any machine with kubectl working):
````
kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 -d > k8s-ca.crt
````
This creates:

````
k8s-ca.crt
````
Step 2: Import cert into Java truststore (on Jenkins server)

Find Java path:

````
readlink -f $(which java)
````
Then run (example path, adjust if needed):
````
sudo keytool -importcert \
  -alias k8s-api \
  -keystore /usr/lib/jvm/java-17-openjdk-amd64/lib/security/cacerts \
  -file k8s-ca.crt
````
Password when asked:

````
changeit
````
type :
````
yes
````
Step 3: Restart Jenkins

````
sudo systemctl restart jenkins
````

========================================================================================

## Why ImagePullBackOff happens
agar aisa erreo aaya toh check karna kubernet konsi image pull kar raha hai 


Usually one of these:

Wrong image name in deployment.yaml

Image not pushed to DockerHub

Private DockerHub repo (needs secret)

Typo in tag (latest vs v1, etc)





