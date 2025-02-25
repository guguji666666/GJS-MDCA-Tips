# List files
```python
import requests
import json
import os
 
FILE_URL = 'https://<tenant name>.<region>.portal.cloudappsecurity.com/api/v1/files'
your_token = '<your token>'
 
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
