``` grrovy
plugins {
    id 'groovy'
}

repositories {
    mavenCentral()
}

dependencies {
    testImplementation 'org.codehaus.groovy:groovy-all:3.0.7'
    testImplementation 'junit:junit:4.13.2'
    testImplementation 'org.jenkins-ci.main:jenkins-core:2.249.2'
    testImplementation 'org.jenkins-ci.plugins:workflow-cps:2.89'
    testImplementation 'org.jenkins-ci.plugins:workflow-step-api:2.22'
    testImplementation 'com.lesfurets:jenkins-pipeline-unit:1.9'
}

----------------------------
def call(String name = 'World') {
    echo "Hello, ${name}!"
}
----------------------------

import com.lesfurets.jenkins.unit.BasePipelineTest
import org.junit.Before
import org.junit.Test

class MyPipelineStepTest extends BasePipelineTest {

    @Before
    void setUp() {
        super.setUp()  // This initializes the testing environment
    }

    @Test
    void testCallStep() {
        // Load and execute the pipeline step
        def script = loadScript("vars/myPipelineStep.groovy")
        
        script.call("Jenkins")  // Call the function with an argument
        assertJobStatusSuccess()  // Verify the job status
        assertEquals("Hello, Jenkins!", helper.callStack.findAll { call -> call.methodName == 'echo' }[0].args[0])
    }
}

````

``` groovy
import groovy.json.JsonOutput
import java.net.HttpURLConnection
import java.net.URL

// Example input string (you can replace this with your actual input)
String inputString = "name:John Doe;age:30;city:New York"

// Split the string by ';' to get each key-value pair
Map<String, String> jsonData = [:]
inputString.split(';').each { pair ->
    // Split each pair by ':' to get key and value
    def (key, value) = pair.split(':')
    jsonData[key] = value
}

// Convert map to JSON
String jsonPayload = JsonOutput.toJson(jsonData)

// Define API endpoint (replace with your actual endpoint)
String apiUrl = "https://example.com/api"

// Send JSON payload to API
def url = new URL(apiUrl)
HttpURLConnection connection = (HttpURLConnection) url.openConnection()
connection.setRequestMethod("POST")
connection.setRequestProperty("Content-Type", "application/json")
connection.setDoOutput(true)

// Write JSON payload to request body
connection.outputStream.withWriter("UTF-8") { writer ->
    writer.write(jsonPayload)
}

// Get response from the API
int responseCode = connection.responseCode
String responseMessage = connection.inputStream.text
println "Response Code: $responseCode"
println "Response Message: $responseMessage"

// Handle errors if needed
if (responseCode != HttpURLConnection.HTTP_OK) {
    println "Error: ${connection.errorStream.text}"
}
```

``` groovy
import groovy.json.JsonBuilder
import groovy.json.JsonSlurper
import groovy.net.HttpURLConnection

// Function to send JSON payload to an API
def sendJsonPayload(String apiUrl, String inputString, String delimiter) {
    // Step 1: Split the input string
    def splitStrings = inputString.split(delimiter)

    // Step 2: Create a JSON payload
    def jsonPayload = new JsonBuilder()
    jsonPayload {
        items splitStrings.collect { item -> item.trim() }
    }

    // Print the JSON payload for debugging
    println "JSON Payload: ${jsonPayload.toPrettyString()}"

    // Step 3: Send the JSON payload to the API
    def url = new URL(apiUrl)
    def connection = url.openConnection() as HttpURLConnection
    connection.requestMethod = 'POST'
    connection.doOutput = true
    connection.setRequestProperty("Content-Type", "application/json")

    // Write the JSON payload to the output stream
    connection.outputStream.withWriter("UTF-8") { writer ->
        writer << jsonPayload.toString()
    }

    // Get the response
    def responseCode = connection.responseCode
    def responseMessage = connection.responseMessage
    println "Response Code: ${responseCode}, Response Message: ${responseMessage}"

    // Optionally, read the response
    if (responseCode == 200) {
        def responseBody = new JsonSlurper().parseText(connection.inputStream.text)
        println "Response Body: ${responseBody}"
    } else {
        println "Error occurred: ${connection.errorStream.text}"
    }

    connection.disconnect()
}

// Example usage
def apiUrl = "https://example.com/api/endpoint"
def inputString = "value1,value2,value3"
def delimiter = ","
sendJsonPayload(apiUrl, inputString, delimiter)

```
