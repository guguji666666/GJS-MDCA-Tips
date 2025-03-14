# List MDA alerts
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
