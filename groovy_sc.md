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
