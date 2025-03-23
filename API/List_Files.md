# List files
## Save results in multiple files
```python
import requests
import json
import os

# URL to the API endpoint from which we want to fetch files/data
FILE_URL = '<Your MDA URL>/api/v1/files/'

# Authorization token to access the API; replace this with your actual token
your_token = '<API token created in XDR portal>'

# Headers for the HTTP request; these usually include authorization and content type
headers = {
    'Authorization': 'Token {}'.format(your_token),
}

# Create output directory to store the fetched files/data, in this sample we use C:\temp\MDCA_files
output_directory = r"C:\temp\MDCA_files" 
try:
    # If the directory doesn't exist, create it. If it does exist, do nothing due to 'exist_ok=True'
    os.makedirs(output_directory, exist_ok=True)
except Exception as e:
    # If directory creation fails, print the error and exit the script
    print(f"Failed to create directory {output_directory}: {e}")
    exit(1)

# Initialize variables for pagination
skipLength = 0  # Number of records to skip, used for paginated requests
has_next = True  # Flag to determine if there is more data to fetch
file_counter = 1  # Counter for the files being saved to ensure unique file names

# Continue fetching data as long as there is more data indicated by `has_next`
while has_next:
    # Define the filters to narrow down the data; commented out part is where specific filters can be applied
    filters = {
        # "limit": 50
        # "policy": {'cabinetmatchedrulesequals': ["62b2e2280fdce234376b03bd"]},
    }

    # Data that needs to be sent in the API request
    request_data = {
        'filters': filters,  # Applying the filters, if any
        'skip': int(skipLength),  # Paginating by specifying how many records to skip
        "limit": 100 # Numbers of results to get each time, max 100
        # 'isScan': True,  # Optionally, some scan-related parameter might be utilized here
    }

    # Sending a POST request to the API with the request data and headers
    response = requests.post(FILE_URL, json=request_data, headers=headers)
    content = json.loads(response.content)  # Parse JSON response from the server
    response_data = content.get('data', [])  # Extract the data portion from the response

    # If no data is returned, print a message and stop fetching further
    if not response_data:
        print("No data returned for this batch.")
        break

    # Define the file path including the current file_counter to save the data uniquely
    file_path = os.path.join(output_directory, f"MDA_files_{file_counter}.json")

    # Open a new file in write mode to save the current batch of data
    with open(file_path, "w") as f:
        # Write the response data to the file in a neatly formatted JSON structure
        json.dump(response_data, f, indent=4)

    # Print the number of records saved and the file path where they were saved
    print('Saved {} records to {}'.format(len(response_data), file_path))

    # Determine if there is more data to fetch based on the API's response
    has_next = content.get('hasNext', False)
    skipLength += 100  # Increment the skip count for the next batch
    file_counter += 1  # Increment file counter for the next file to be unique

# Print summary message after all data fetching is completed
print('Finished fetching data.')
```

## Save results in single file
```python
import requests

import json

import os

FILE_URL = '<Your MDA URL>/api/v1/files/'

your_token = '<API token created in XDR portal>'

headers = {

    'Authorization': 'Token {}'.format(your_token),

}

# Create output directory if it doesn't exist

output_directory = r"C:\temp\MDCA_files"

try:

    os.makedirs(output_directory, exist_ok=True)

except Exception as e:

    print(f"Failed to create directory {output_directory}: {e}")

    exit(1)

# Open the output file in write mode

file_path = os.path.join(output_directory, "mdca_files.json")

f = open(file_path, "w")

skipLength = 0

records = []

has_next = True

n = 0

# Write the opening bracket for the JSON array

f.write('[')

while has_next:

    filters = {

        # "policy": {'cabinetmatchedrulesequals': ["62b2e2280fdce234376b03bd"]},

    }

    request_data = {

        'filters': filters,

        'skip': str(skipLength),

        # 'isScan': True,

    }

    response = requests.post(FILE_URL, json=request_data, headers=headers)

    content = json.loads(response.content)

    response_data = content.get('data', [])

    # print(response_data)

    records += response_data

    # Convert the response_data to a JSON string

    batch_json = json.dumps(response_data)

    # Write the batch_json to the file

    # If it's not the first batch, prepend a comma

    if len(records) > len(response_data):
        f.write(',')

    f.write(batch_json)

    print('Got {} more records'.format(len(response_data)))

    has_next = content.get('hasNext', False)

    skipLength += 100

# Close the JSON array

f.write(']')

print('Got {} records in total'.format(len(records)))

f.close()
```
