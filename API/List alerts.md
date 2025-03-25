# List MDA alerts

## Powershell scripts
### 1. List alerts once ( 100 results Max returned by design)
```powershell
# Define the necessary variables
$tokenKey = "<API token created in XDR portal>"

# Specify the full API endpoint URL directly
$url = "<Your MDA url>/api/v1/alerts/"


# Define the headers
$headers = @{
    "Authorization" = "Token $tokenKey"
    "Content-Type"  = "application/json"
}

# Create the JSON body
$body = @{
    filters = @{
        # Add filter criteria here
    }
    skip    = 5
    limit   = 10
    # Add other parameters if needed
} | ConvertTo-Json

# Make the POST request using Invoke-RestMethod
$response = Invoke-RestMethod -Method Post -Uri $url -Headers $headers -Body $body

# Convert the response to a pretty-printed JSON format
$formattedJson = $response | ConvertTo-Json -Depth 10 -Compress

# Define the output file path
$outputFilePath = ".\MDAalerts.json"

# Save the formatted JSON to a file
Set-Content -Path $outputFilePath -Value $formattedJson -Encoding utf8

# Output the file path for confirmation
Write-Output "Output saved to $outputFilePath"
```

## Javascript
### 1. List alerts in loop with specified timerange, also format time from Unix to user friendly view
```powershell
npm init -y

npm install axios

ls node_modules
```
![image](https://github.com/user-attachments/assets/13b6053a-325b-4b06-a3b5-77d26583da67)


```js
const axios = require('axios');

// Function to convert Unix timestamp (in milliseconds) to formatted local time
function formatUnixTimestamp(timestamp) {
    const date = new Date(timestamp); // Convert milliseconds to Date object
    return {
        TimestampFormatted: date.toISOString().slice(0, 19).replace('T', ' '), // Format date as string
        LocalTime: date // Keep the original Date object
    };
}

// Async function to handle API requests and data processing
async function fetchAndProcessAlerts() {
    const API_URL = '<Your XDR URL>/api/v1/alerts/';
    const API_TOKEN = 'Token <Your Token>'; // Replace with your actual MDA API Token
    
    let skip = 0;
    const limit = 100;
    let allAlerts = [];

    while (true) {
        const body = {
            "skip": skip,
            "limit": limit,
            "sortField": "date",
            "sortDirection": "desc"
        };

        try {
            console.log(`Retrieving alert data: skip = ${skip}, limit = ${limit}.`);
            
            const response = await axios.post(API_URL, body, {
                headers: {
                    'Authorization': API_TOKEN,
                    'Content-Type': 'application/json'
                }
            });

            if (response.data.data.length === 0) break; 
            
            allAlerts = allAlerts.concat(response.data.data);
            skip += limit;
        } catch (error) {
            console.error('Failed to fetch alerts:', error.message);
            break;
        }
    }

    // Format Unix Timestamps for all alerts
    allAlerts = allAlerts.map(alert => ({
        ...alert,
        ...formatUnixTimestamp(alert.timestamp)
    }));

    const startDate = new Date('2025-03-24T03:00:00Z'); //Input start UTC time
    const endDate = new Date('2025-03-24T23:59:59Z'); //Input End UTC time 

    // Filter alerts within the time range
    const filteredAlerts = allAlerts.filter(alert => 
        alert.LocalTime >= startDate && alert.LocalTime <= endDate
    );

    console.log(`Total alerts found: ${allAlerts.length}`);
    console.log(`Alerts within specified time range: ${filteredAlerts.length}`);

    // Log each alert with the desired format
    filteredAlerts.forEach(alert => {
        console.log(`_id                   : ${alert._id}`);
        console.log(`timestamp             : ${alert.timestamp}`);
        console.log(`entities              : ${alert.entities.map(entity => JSON.stringify(entity)).join(', ')}...`);
        console.log(`title                 : ${alert.title}`);
        console.log(`description           : ${alert.description}`);
        console.log(`stories               : ${alert.stories ? `{${alert.stories.join(', ')}}` : '{}'}`);
        console.log(`policy                : @{${Object.entries(alert.policy).map(([key, value]) => `${key}=${value}`).join('; ')}}`);
        console.log(`contextId             : ${alert.contextId}`);
        console.log(`intent                : ${alert.intent ? `{${alert.intent.join(', ')}}` : '{}'}`);
        console.log(`idValue               : ${alert.idValue}`);
        console.log(`statusValue           : ${alert.statusValue}`);
        console.log(`severityValue         : ${alert.severityValue}`);
        console.log(`resolutionStatusValue : ${alert.resolutionStatusValue}`);
        console.log(`isSystemAlert         : ${alert.isSystemAlert ? 'True' : 'False'}`);
        console.log(`URL                   : ${alert.URL}`);
        console.log(`TimestampFormatted    : ${alert.TimestampFormatted}`);
        console.log(`LocalTime             : ${alert.LocalTime.toLocaleString()}`);
        console.log(''); // Add a blank line for readability
    });
}

// Run the script
fetchAndProcessAlerts().catch(console.error);
```

### Run Javascript
```powershell
cd <path to your javascript>

node .\<name of script>.js
```

Sample output <br>
![image](https://github.com/user-attachments/assets/2f8f6cd0-b4b8-4742-8ea7-47d8cf631747)
