``` groovy
@Grab(group='com.jcraft', module='jsch', version='0.1.55')
import com.jcraft.jsch.JSch
import com.jcraft.jsch.ChannelExec
import com.jcraft.jsch.Session

// Define the remote connection parameters
def remoteHost = 'your.remote.server'     // Remote server IP or hostname
def username = 'remoteUser'               // Username for SSH
def password = 'remotePassword'           // Password for SSH (or use a private key)

// Shell script command to execute on the remote server
def shellScriptCommand = 'bash /path/to/your/script.sh'  // Update to the path of your script

// Function to execute shell script on the remote server
def executeRemoteShellCommand(String host, String user, String password, String command) {
    def output = new StringBuilder()
    def jsch = new JSch()
    Session session = null
    ChannelExec channel = null
    
    try {
        // Initialize the SSH session
        session = jsch.getSession(user, host, 22)
        session.setPassword(password)
        session.setConfig("StrictHostKeyChecking", "no")
        session.connect()

        // Create the command execution channel
        channel = session.openChannel("exec") as ChannelExec
        channel.setCommand(command)
        
        // Capture the output and errors from the remote command execution
        def inputStream = channel.inputStream
        def errorStream = channel.errStream
        
        // Connect and execute the command
        channel.connect()

        // Read the output from the command
        inputStream.eachLine { output.append(it).append('\n') }

        // Check for errors (optional)
        if (channel.exitStatus != 0) {
            errorStream.eachLine { output.append("ERROR: ").append(it).append('\n') }
        }

    } catch (Exception e) {
        println "Exception: ${e.message}"
    } finally {
        // Clean up the resources
        if (channel != null) channel.disconnect()
        if (session != null) session.disconnect()
    }

    return output.toString()
}

// Test the remote shell script execution
def testRemoteShellExecution() {
    println "Testing connection and shell script execution..."
    def result = executeRemoteShellCommand(remoteHost, username, password, shellScriptCommand)

    // Simple validation for testing if the output contains expected results
    if (result.contains('expected output from script')) {
        println "Test Passed: Script executed successfully and produced the expected output."
    } else {
        println "Test Failed: Script execution didn't return expected output."
    }

    println "Remote Shell Script Output:\n$result"
}

// Call the test function to execute and validate
testRemoteShellExecution()

```

``` sh
#!/bin/bash

# Define the location for profiles and default configuration values
PROFILES_DIR="/path/to/nbu/profiles"
NBU_VERSION="9.1" # Set your NBU target version here
UPGRADE_DATE=$(date +"%Y-%m-%d")
CURRENT_DATE=$(date +"%Y-%m-%d %H:%M:%S")

# Function to create an upgrade profile
create_profile() {
    local server_name="$1"
    local scheduled_time="$2"
    local profile_path="$PROFILES_DIR/$server_name"

    # Check if the profile already exists
    if [ -d "$profile_path" ]; then
        echo "Profile for server $server_name already exists. Skipping..."
        return 0
    fi

    # Check if the scheduled time is in the future
    if [[ "$CURRENT_DATE" > "$scheduled_time" ]]; then
        echo "Scheduled time for $server_name has already passed. Skipping..."
        return 0
    fi

    echo "Creating upgrade profile for server: $server_name"

    # Create a directory for the server profile
    mkdir -p "$profile_path" || { echo "Failed to create profile directory for $server_name"; return 1; }

    # Create a profile configuration file
    cat <<EOF > "$profile_path/upgrade_profile.conf"
# NetBackup Media Server Upgrade Profile
server_name=$server_name
nbu_version=$NBU_VERSION
upgrade_date=$UPGRADE_DATE
scheduled_time=$scheduled_time
EOF

    echo "Upgrade profile created at $profile_path/upgrade_profile.conf"
}

# Main script
if [ "$#" -lt 2 ]; then
    echo "Usage: $0 server1 scheduled_time1 [server2 scheduled_time2 ...]"
    echo "Scheduled time format: YYYY-MM-DD HH:MM:SS"
    exit 1
fi

# Ensure the profiles directory exists
mkdir -p "$PROFILES_DIR" || { echo "Failed to create profiles directory at $PROFILES_DIR"; exit 1; }

# Loop through each server and its scheduled time
while [ "$#" -gt 0 ]; do
    server="$1"
    scheduled_time="$2"
    create_profile "$server" "$scheduled_time"
    shift 2
done

echo "Profile creation process completed."

```
