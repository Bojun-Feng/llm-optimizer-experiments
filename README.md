Here is the updated tutorial tailored to your specific hardware, software, and safety requirements.

This guide provides the scripts to create a self-terminating Google Cloud instance with an `a2-highgpu-8g` machine type, ensuring your experiment automatically shuts down after a maximum of two hours to protect your budget.

### **Important Note on Data Persistence**

You have requested Local SSDs for maximum performance. It is critical to understand that **Local SSDs are ephemeral**. All data stored on them will be **permanently lost** when the instance is stopped or deleted. Your startup script is designed to delete the instance after two hours. You must ensure your PyTorch script saves its final model checkpoints, logs, and any other results to a persistent location (like a Google Cloud Storage bucket or by transferring it off the machine) *before* the two-hour limit is reached.

---

### Protocol 1: Starting Your Self-Terminating Compute Node

This process involves two parts: first, creating a small shutdown script, and second, launching the VM instance with instructions to run that script on startup.

**Step 1: Create the Auto-Shutdown Script**

This script will run in the background on your new VM. It waits for two hours and then calls the command to delete the instance it is running on.

Create a file on your local machine named `shutdown_script.sh` and paste the following content into it:

```bash
#!/bin/bash

# This script is executed by the VM on startup.
# It launches a background process that will forcefully delete the VM after 2 hours.

(
  echo "Instance will self-destruct in 2 hours."
  # Wait for 2 hours (7200 seconds)
  sleep 7200

  echo "Time limit reached. Deleting instance now..."
  # Get instance name and zone from the metadata server to delete itself
  INSTANCE_NAME=$(curl -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/name)
  INSTANCE_ZONE=$(curl -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/zone | cut -d'/' -f4)

  # Command to delete the current instance
  gcloud compute instances delete "$INSTANCE_NAME" --zone="$INSTANCE_ZONE" --quiet

) & # The '&' runs this whole block in the background, allowing your main work to proceed.
```

**Step 2: Create the VM Instance**

Now, run the following command from your terminal. It will create the powerful A2 instance, attach the Local SSDs, and pass the `shutdown_script.sh` to be executed upon boot.

Make sure the `shutdown_script.sh` file is in the same directory where you run this command.

```bash
#!/bin/bash

# --- Configuration ---
export INSTANCE_NAME="nanogpt-a2-autoterm"
export ZONE="us-central1-a"  # Iowa region, zone 'a'. Check availability if needed.
export MACHINE_TYPE="a2-highgpu-8g" # 8x A100 GPUs
export IMAGE_FAMILY="pytorch-latest-gpu" # PyTorch on Ubuntu with necessary drivers
export IMAGE_PROJECT="deeplearning-platform-release"
export BOOT_DISK_TYPE="pd-standard" # Standard persistent disk for the OS, as requested
export BOOT_DISK_SIZE="100GB"

# --- Create the VM Instance with the Self-Destruct Timer ---
echo "Creating instance: $INSTANCE_NAME..."
echo "It will automatically terminate after 2 hours."

gcloud compute instances create $INSTANCE_NAME \
    --zone=$ZONE \
    --machine-type=$MACHINE_TYPE \
    --maintenance-policy=TERMINATE \
    --image-family=$IMAGE_FAMILY \
    --image-project=$IMAGE_PROJECT \
    --local-ssd=interface=NVME \
    --boot-disk-type=$BOOT_DISK_TYPE \
    --boot-disk-size=$BOOT_DISK_SIZE \
    --boot-disk-auto-delete \
    --scopes=https://www.googleapis.com/auth/cloud-platform \
    --metadata-from-file=startup-script=shutdown_script.sh

echo "Instance $INSTANCE_NAME created."
echo "IMPORTANT: This instance will be automatically deleted in 2 hours."
echo "To connect, run: gcloud compute ssh --zone $ZONE $INSTANCE_NAME"
```

**Explanation of Key Changes:**

*   **`--machine-type="a2-highgpu-8g"`**: Selects the instance with 8 NVIDIA A100 GPUs.
*   **`--zone="us-central1-a"`**: Places the machine in the Iowa region.
*   **`--local-ssd=interface=NVME`**: Attaches the high-performance Local SSDs. The `a2-highgpu-8g` type automatically includes 8x375 GB disks with this single flag.
*   **`--boot-disk-type="pd-standard"`**: Ensures the operating system resides on a lower-cost standard hard drive.
*   **`--metadata-from-file=startup-script=shutdown_script.sh`**: This is the built-in safety mechanism. It tells Google Cloud to take your `shutdown_script.sh` file and execute it the first time the VM boots up.

---

### Protocol 2: Shutting Down Manually (Early Completion Override)

If your job finishes in less than two hours, you don't need to wait for the automatic shutdown. Running this script will immediately delete the instance and stop the billing.

**The Manual Deletion Script:**

```bash
#!/bin/bash

# --- Configuration ---
# Ensure this matches the instance you created
export INSTANCE_NAME="nanogpt-a2-autoterm"
export ZONE="us-central1-a"

# --- Delete the VM Instance ---
echo "Manually deleting instance: $INSTANCE_NAME and its disks..."
gcloud compute instances delete $INSTANCE_NAME \
    --zone=$ZONE \
    --quiet

echo "Instance $INSTANCE_NAME and all its resources have been successfully deleted."
```

---

### Part 1: One-Time Setup on Your LOCAL Computer

You only need to do this once. The goal is to create an `rclone.conf` file that acts as a secure key to your Google Drive.

**Step 1: Install rclone**
Follow the official instructions for your operating system: [rclone.org/install/](https://rclone.org/install/)
*   **macOS/Linux:** `sudo -v ; curl https://rclone.org/install.sh | sudo bash`
*   **Windows:** Download the correct `.zip` file and add the location of `rclone.exe` to your system's PATH.

**Step 2: Configure rclone**
Open a terminal on your local machine and run:
`rclone config`

You will be guided through an interactive setup. Here’s exactly what to enter:

1.  `n` (for New remote)
2.  `name>`: Enter `gdrive` (or any name you prefer)
3.  `storage>`: Type `drive` or find "Google Drive" in the list and enter its number.
4.  `client_id>`: Press **Enter** to leave blank.
5.  `client_secret>`: Press **Enter** to leave blank.
6.  `scope>`: Enter `1` for full access (`drive`).
7.  `root_folder_id>`: Press **Enter** to leave blank. We will specify the folder during the copy command.
8.  `service_account_file>`: Press **Enter** to leave blank.
9.  `Edit advanced config?`: Enter `n` (No).
10. `Use auto config?`: Enter `y` (Yes).

At this point, your web browser should open, asking you to log in to your Google Account and grant `rclone` permission. **Log in and click "Allow"**.

11. Back in your terminal, it will ask: `Configure this as a team drive?`: Enter `n` (No).
12. You'll see a summary. If it looks correct, enter `y` (Yes).
13. Finally, enter `q` to quit the configuration.

**Step 3: Locate Your Secret Configuration File**
You have now created a file named `rclone.conf`. This file is your key. **Treat it like a password.** Find it in the following location:
*   **macOS/Linux:** `~/.config/rclone/rclone.conf`
*   **Windows:** `C:\Users\<YourUsername>\AppData\Roaming\rclone\rclone.conf`

Keep this file safe and accessible on your local machine.

---

### Part 2: Workflow for Each Cloud Experiment

Now, here is the repeatable process for every time you run your experiment on a new VM.

**Step 1: Find Your Google Drive Folder ID**
In your web browser, navigate to the specific folder in Google Drive where you want to save your output. Look at the URL in the address bar. The ID is the last part of the URL.

For example, if the URL is `https://drive.google.com/drive/folders/1aBcDeFgHiJkLmNoPqRsTuVwXyZ_12345`, your Folder ID is `1aBcDeFgHiJkLmNoPqRsTuVwXyZ_12345`.

Copy this ID and save it.

**Step 2: Launch and Connect to your VM**
Use the startup script from our previous conversation to create your self-terminating `a2-highgpu-8g` instance. Once it's running, connect to it via SSH:
`gcloud compute ssh --zone us-central1-a nanogpt-a2-autoterm`

**Step 3: Upload Your `rclone.conf` File to the VM**
Open a **new, separate terminal window** on your local machine. Use the `gcloud compute scp` command to securely copy your configuration file to the VM's home directory.

*   **On macOS/Linux:**
    ```bash
    gcloud compute scp ~/.config/rclone/rclone.conf nanogpt-a2-autoterm:~/ --zone=us-central1-a
    ```
*   **On Windows:**
    ```bash
    gcloud compute scp "C:\Users\<YourUsername>\AppData\Roaming\rclone\rclone.conf" nanogpt-a2-autoterm:~/ --zone=us-central1-a
    ```

**Step 4: Create an Automated "Run and Upload" Script**
Now, back in the SSH session on your VM, we will create a master script that runs your experiment, uploads the results, and then safely shuts down the machine.

Create a file named `run_and_upload.sh` on the VM:
`nano run_and_upload.sh`

Paste the following content into the editor. **Make sure to change the placeholder values.**

```bash
#!/bin/bash
# This script automates the entire experiment lifecycle.

# Exit immediately if any command fails
set -e

# --- Configuration ---
# 1. The ID of the specific Google Drive folder you copied earlier
GDRIVE_FOLDER_ID="1aBcDeFgHiJkLmNoPqRsTuVwXyZ_12345"

# 2. The path to the output folder your script will generate
OUTPUT_FOLDER_PATH="./output" # Change if your script saves elsewhere

# 3. A name for the folder as it will appear in Google Drive (can include date)
UPLOAD_FOLDER_NAME="nanogpt-run-$(date +%Y-%m-%d-%H%M)"

echo "Step 1: Preparing environment..."
# Install rclone on the VM
sudo apt-get update && sudo apt-get install -y rclone

# Move the configuration file to the correct location
mkdir -p ~/.config/rclone
mv ~/rclone.conf ~/.config/rclone/

echo "Step 2: Starting PyTorch training script..."
# --- RUN YOUR ACTUAL EXPERIMENT HERE ---
# This command will block until your Python script finishes
python3 your_nanogpt_training_script.py
echo "Training finished."

echo "Step 3: Uploading results to Google Drive..."
# Command to copy the local output folder to your specific Google Drive folder
# It creates a new sub-folder with the UPLOAD_FOLDER_NAME
rclone copy "$OUTPUT_FOLDER_PATH" "gdrive:/$UPLOAD_FOLDER_NAME" --drive-root-folder-id "$GDRIVE_FOLDER_ID" --progress

echo "Upload complete! Results are in Google Drive."

echo "Step 4: Shutting down instance..."
# Get instance name and zone to self-destruct
INSTANCE_NAME=$(curl -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/name)
INSTANCE_ZONE=$(curl -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/zone | cut -d'/' -f4)

# Command to delete the current instance
gcloud compute instances delete "$INSTANCE_NAME" --zone="$INSTANCE_ZONE" --quiet
```

Save the file (`Ctrl+O`, then `Enter`) and exit nano (`Ctrl+X`).

**Step 5: Run Your Experiment**
Make the script executable and run it:
```bash
chmod +x run_and_upload.sh
./run_and_upload.sh
```

Now you can sit back. The script will:
1.  Set up the environment.
2.  Run your Python code.
3.  Once finished, automatically upload the specified output folder to your designated Google Drive folder.
4.  Delete the instance to stop all billing.

This method is secure, fully automates the cleanup and data transfer, and is the easiest way to achieve your goal.
