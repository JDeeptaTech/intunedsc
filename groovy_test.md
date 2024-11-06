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
