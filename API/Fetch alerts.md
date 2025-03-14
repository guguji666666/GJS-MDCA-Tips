# Fetch MDA alert
```powershell
# Define the token key used for API authentication
$tokenKey = "<API token created in XDR portal>"

# Define the specific PK value to be used in the API request URL
$pk = "<MDA alert id>" # Specify your PK value here

# Construct the full API endpoint URL directly, incorporating the PK into the path
$url = "<MDA url>/api/v1/alerts/$pk/"

# Define the headers required for the API request, including authentication
$headers = @{
    "Authorization" = "Token $tokenKey"  # Authorization token for authentication
    "Content-Type"  = "application/json" # Content type of the request
}

# Create the JSON body if needed (generally not used for a GET request)
$body = @{
    # Add parameters to the JSON body if needed for the request
} | ConvertTo-Json

# Execute the GET request to the API endpoint using Invoke-RestMethod
$response = Invoke-RestMethod -Method Get -Uri $url -Headers $headers

# Convert the API response into a formatted JSON string
$formattedJson = $response | ConvertTo-Json -Depth 10 -Compress

# Define the output file path, dynamically naming the file using PK
$outputFilePath = ".\MDA_Alert_$pk.json"

# Save the formatted JSON response to the file with the dynamic filename
Set-Content -Path $outputFilePath -Value $formattedJson -Encoding utf8

# Output the path of the saved file to confirm successful operation
Write-Output "Output saved to $outputFilePath"

```
