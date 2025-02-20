# List defender for cloud apps activities
```python
import requests  # Import the requests library to handle HTTP requests
import json  # Import the json library to parse and work with JSON data
import os  # Import the os library to interact with the operating system, such as file paths and directories

# Define the URL to which we will make the requests for activities
ACTIVITIES_URL = '<your url>/api/v1/activities/'

# Set your token for authorization; replace with your actual token
your_token = '<your token>'

# Create headers for the request, including the authorization token
headers = {
    'Authorization': 'Token {}'.format(your_token),
}

# Define filters for the request. You can modify these filters as needed.
filters = {
    # This filter specifies to retrieve records from the last 14 days
    'date': {'gte_ndays': 7},
    # 'service': {'eq': [20893]}
}

# Prepare the body of the request containing the filters and a flag indicating if it is a scan
request_data = {
    'filters': filters,
    'isScan': True,  # Enable scanning mode, which allows for pagination of results
    "limit": 4200,  # Uncomment to limit the number of records per request
}

# Initialize a list to store the retrieved records
records = []
# Flag to check if there are more pages of results to fetch
has_next = True
# Counter for the output file names
file_counter = 1

# Ensure the output directory exists
output_directory = r'C:\temp\MDCA_Activities'
os.makedirs(output_directory, exist_ok=True)

# Loop to perform requests and retrieve all relevant records
while has_next:
    # Send a POST request to the API to fetch activity data
    response = requests.post(ACTIVITIES_URL, json=request_data, headers=headers)

    # Parse the JSON content of the response
    content = json.loads(response.content)

    # Extract the data from the response, defaulting to an empty list if 'data' is not present
    response_data = content.get('data', [])

    # Append the retrieved records to the records list
    records += response_data
    print('Got {} more records'.format(len(response_data)))  # Print the number of records retrieved in this batch

    # Write the current batch of records to a JSON file
    output_file_path = os.path.join(output_directory, f'output{file_counter}.json')
    with open(output_file_path, 'w') as json_file:
        json.dump(response_data, json_file, indent=4)  # Save records to file with pretty JSON formatting
        print(f'Written {len(response_data)} records to {output_file_path}')  # Print confirmation of write

    # Increment the counter for the next output file name
    file_counter += 1

    # Retrieve the hasNext flag from the response to check if there are more records
    has_next = content.get('hasNext', False)
    print('Has next page: {}'.format(has_next))  # Print whether there are more pages available

    # Retrieve any next query filters provided by the response
    next_query_filters = content.get('nextQueryFilters', {})
    print('Next query filters: {}'.format(next_query_filters))  # Print the next query filters for the following request

    # Update the filters in the request data to use the next query filters if available
    request_data['filters'] = next_query_filters

# After exiting the loop, print the total number of records retrieved
print('Got {} records in total'.format(len(records)))
```
