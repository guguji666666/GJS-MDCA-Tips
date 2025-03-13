# List risky apps detected per user
## 1.Find `streamId`
![image](https://github.com/user-attachments/assets/9a0da083-d75e-4852-8df4-ad344ca91cce)

## 2.Export all users, rename to `MDCAUsers.csv`
![image](https://github.com/user-attachments/assets/3a19b853-0185-4adb-b403-2e96d25f7920)

![image](https://github.com/user-attachments/assets/0d9de65f-10e8-4716-bb2f-af11575257f3)

## 3.Create API Token in XDR portal
![image](https://github.com/user-attachments/assets/7b05f3d0-cbe7-4e13-9799-24da5c42b724)

## 4.Find your MDA URL
![image](https://github.com/user-attachments/assets/2ae7aa8e-3ea9-4ca9-9fc2-91d6dba90433)

## Python code
```python
# Script to fetch discovered applications for users listed in a CSV file from the MDA (Mobile Device Analytics) portal
# and save the results to user-specific JSON files.
#
# ### Prerequisites and Parameters
# Before running this script, ensure you have the following parameters and files prepared:
#
# 1. **DISCOVERY_URL**:
#    - Description: The base URL for the API endpoint to fetch discovered applications.
#    - How to Obtain: Replace `<Your MDA url>` with the actual URL of your MDA portal, typically provided by your
#      administrator or found in the MDA portal documentation. The full URL should look like:
#      `https://<your-mda-domain>/cas/api/v1/discovery/discovered_apps/`.
#
# 2. **TOKEN**:
#    - Description: The API token used for authentication.
#    - How to Obtain: Generate an API token in the XDR (Extended Detection and Response) portal under your account settings
#      or API management section. Replace `<API token created in XDR portal>` with the actual token.
#
# 3. **streamId**:
#    - Description: A unique identifier for the data stream, required in the API request payload.
#    - How to Obtain: This can be obtained by performing a HAR (HTTP Archive) trace while accessing the MDA portal.
#      Steps:
#      - Log in to the MDA XDR portal.
#      - Navigate to Cloud Apps > Cloud Discovery > Users.
#      - Select a user and view their Discovered Apps.
#      - Use browser developer tools (e.g., Chrome DevTools) to capture network traffic, export it as a HAR file, and
#        search for the `streamId` in the API requests (e.g., `689bad7bc21bc69bae98b98f`).
#
# 4. **MDCAUsers.csv**:
#    - Description: A CSV file containing user IDs in a column named `Name`.
#    - How to Obtain: Download this file from the MDA portal.
#      Steps:
#      - Log in to the MDA portal.
#      - Navigate to Cloud Discovery > Users.
#      - Click the "Export all users" option to download the CSV file.
#    - File Placement: Save the `MDCAUsers.csv` file in the same directory as this Python script.
#
# 5. **Output Files**:
#    - Description: The script generates user-specific JSON files named `<user_id>_discovered_apps.json` for each user.
#    - File Placement: These output files will be saved in the same directory as the Python script.
#
# 6. **Score Range Filter**:
#    - Description: The script filters applications based on a risk score range, currently set to 3-10 (inclusive).
#    - Customization: Modify the `score` filter in the `get_request_payload` function if you need a different range.
#      For example, `"eq": [1, 5]` would filter for scores between 1 and 5.
#
# ### Usage
# 1. Ensure all prerequisites are met (parameters set and `MDCAUsers.csv` in place).
# 2. Run the script using Python: `python script_name.py`.
# 3. Monitor the console for logging messages to track progress and identify any errors.
# 4. Check the output directory for the generated JSON files.
#
# ### Dependencies
# This script requires the following Python libraries, which can be installed using pip:
# - requests: For making HTTP requests to the API.
# - json: For handling JSON data (built-in).
# - datetime: For timestamp generation (built-in).
# - logging: For logging messages (built-in).
# - sys: For system-specific parameters and functions (built-in).
# - csv: For reading CSV files (built-in).
# - os: For file path operations (built-in).
# - time: For introducing delays between batches (built-in).
#
# Install the `requests` library if not already installed:
# ```bash
# pip install requests
# ```
#
# ### Batch Processing
# The script processes user IDs in batches of 100, fetching user IDs from the CSV, generating JSON output files one by one
# for each user in the batch, waiting 60 seconds after completing the batch, and then continuing with the next batch of  100 user IDs until all users are processed.


import requests
import json
from datetime import datetime
import logging
import sys
import csv
import os
import time  # Added for sleep functionality

# Configure basic logging to display informational and error messages
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Constants for API configuration
DISCOVERY_URL = '<your MDA URL>/cas/api/v1/discovery/discovered_apps/'
TOKEN = '<API token created in XDR portal>'

HEADERS = {
    'Authorization': f'Token {TOKEN}',  # Authorization header using the API token
    'Content-Type': 'application/json'  # Specifies the content type of the request body
}


def get_request_payload(user_id):
    """
    Constructs and returns the initial request payload for the API call with the specified user ID.

    Args:
        user_id (str): The unique identifier for the user to filter the API results.

    Returns:
        dict: A dictionary containing the payload with pagination, filters, sorting, and timeframe settings.
    """
    return {
        "skip": 0,  # Starting index for pagination (initially 0)
        "limit": 100,  # Maximum number of records to fetch per API call
        "filters": {
            "user.entity": {
                "eq": [user_id]  # Filter to match records for the specific user ID
            },
            "score": {
                "eq": [1, 5]  # Filter to include only records with risk scores between 3 and 10 (inclusive)
            }
        },
        "sortField": "score",  # Field to sort the results by (in this case, risk score)
        "sortDirection": "desc",  # Sorting direction (descending order, highest risk first)
        "streamId": "67c80bd061d024c890fe59c5",  # Unique identifier for the data stream, you could get it from har tracing
        "timeframe": "30",  # Timeframe for data retrieval (last 30 days)
        "performAsyncTotal": True  # Flag to enable asynchronous total count calculation
    }


def fetch_discovered_apps(user_id):
    """
    Fetches discovered applications from the API for the given user ID and processes the results.

    Args:
        user_id (str): The unique identifier for the user to fetch applications for.

    Returns:
        tuple: A tuple containing:
            - list: All fetched records from the API.
            - list: Sorted list of unique application names.

    Raises:
        requests.RequestException: If the API request fails.
        json.JSONDecodeError: If the API response cannot be decoded as JSON.
    """
    records = []  # List to store all fetched records
    app_names = set()  # Set to store unique application names (avoids duplicates)
    request_data = get_request_payload(user_id)  # Initialize the request payload
    has_next = True  # Flag to control pagination loop

    try:
        while has_next:
            # Make a POST request to the API with the current payload and headers
            response = requests.post(
                DISCOVERY_URL,
                json=request_data,
                headers=HEADERS
            )
            response.raise_for_status()  # Raise an exception for HTTP errors (e.g., 4xx, 5xx)

            # Parse the JSON response from the API
            content = response.json()
            response_data = content.get('data', [])  # Extract the data field (list of records)
            records.extend(response_data)  # Add new records to the master list

            # Extract application names from the current batch of records
            for item in response_data:
                app_names.add(item['name'])  # Add app name to the set (duplicates are automatically ignored)

            logger.info(f'Fetched {len(response_data)} new records. Total: {len(records)}')

            # Check if there are more records to fetch (pagination)
            has_next = content.get('hasNext', False)  # Update the flag based on API response
            request_data['filters'] = content.get('nextQueryFilters', {})  # Update filters for the next request

        return records, sorted(app_names)  # Return all records and sorted list of unique app names

    except requests.RequestException as e:
        logger.error(f"Error fetching data from API: {str(e)}")
        raise
    except json.JSONDecodeError as e:
        logger.error(f"Error decoding JSON response: {str(e)}")
        raise


def save_results(app_names, records, user_id):
    """
    Saves the fetched results to a user-specific JSON file.

    Args:
        app_names (list): Sorted list of unique application names.
        records (list): List of all records fetched from the API.
        user_id (str): The user ID associated with the results.

    Raises:
        IOError: If there is an error writing to the output files.
    """
    try:
        # Prepare the structured JSON output
        result = {
            "timestamp": datetime.now().isoformat(),  # Current timestamp in ISO format
            "user_id": user_id,  # The user ID associated with the results
            "total_records": len(records),  # Total number of records fetched
            "applications": [
                {
                    "name": name,  # Application name
                    "count": sum(1 for r in records if r['name'] == name)  # Number of occurrences of this app
                } for name in app_names
            ]
        }

        # Generate a dynamic JSON filename based on the user ID
        json_output_file = f"{user_id}_discovered_apps.json"

        # Save the structured results to a JSON file
        with open(json_output_file, 'w', encoding='utf-8') as f:
            json.dump(result, f, indent=2)  # Write JSON with indentation for readability

        logger.info(f"Results saved to {json_output_file}")

    except IOError as e:
        logger.error(f"Error writing to file: {str(e)}")
        raise


def read_user_ids_from_csv(csv_file_path, start_index, batch_size):
    """
    Reads a batch of user IDs from the 'Name' column of a CSV file starting at the specified index.

    Args:
        csv_file_path (str): Path to the CSV file (MDCAUsers.csv).
        start_index (int): The starting index to read user IDs from.
        batch_size (int): The number of user IDs to read in this batch.

    Returns:
        tuple: A tuple containing:
            - list: List of user IDs for the current batch.
            - int: Total number of user IDs in the CSV file.

    Raises:
        FileNotFoundError: If the CSV file does not exist.
        csv.Error: If there is an error reading the CSV file.
        KeyError: If the 'Name' column is not found in the CSV.
    """
    user_ids = []
    total_user_ids = 0
    try:
        with open(csv_file_path, 'r', encoding='utf-8') as csvfile:
            reader = csv.DictReader(csvfile)  # Read CSV as a dictionary with column headers
            if 'Name' not in reader.fieldnames:
                raise KeyError("Column 'Name' not found in CSV file")

            # Convert reader to list to count total rows and slice for batch
            all_rows = list(reader)
            total_user_ids = len(all_rows)

            # Calculate the end index for the current batch
            end_index = min(start_index + batch_size, total_user_ids)

            # Extract user IDs for the current batch
            for i in range(start_index, end_index):
                user_ids.append(all_rows[i]['Name'])

        logger.info(f"Read {len(user_ids)} user IDs for batch starting at index {start_index} from {csv_file_path}")
        return user_ids, total_user_ids
    except FileNotFoundError:
        logger.error(f"CSV file not found: {csv_file_path}")
        raise
    except csv.Error as e:
        logger.error(f"Error reading CSV file: {str(e)}")
        raise


def main():
    """
    Main function to orchestrate the application discovery process for all user IDs in the CSV file.
    Iterates over user IDs in batches of 100, fetches their discovered applications, saves the results one by one,
    waits 60 seconds between batches, and continues until all users are processed.
    """
    try:
        # Determine the path to the CSV file (same directory as this script)
        script_dir = os.path.dirname(os.path.abspath(__file__))  # Get the directory of the current script
        csv_file_path = os.path.join(script_dir, 'MDCAUsers.csv')  # Construct path to MDCAUsers.csv

        # Define batch size and delay
        batch_size = 30  # Process 100 user IDs at a time
        delay_seconds = 30  # Wait 60 seconds between batches

        start_index = 0  # Starting index for reading user IDs
        total_user_ids = 0  # Total number of user IDs in the CSV

        while True:
            # Read the next batch of user IDs
            batch_user_ids, total_user_ids = read_user_ids_from_csv(csv_file_path, start_index, batch_size)

            if not batch_user_ids:  # If no more user IDs to process, break the loop
                logger.info("All user IDs processed successfully.")
                break

            logger.info(
                f"Processing batch {start_index // batch_size + 1}: Users {start_index + 1} to {start_index + len(batch_user_ids)} of {total_user_ids}")

            # Process each user ID in the current batch
            for user_id in batch_user_ids:
                logger.info(f"Starting application discovery process for user: {user_id}")

                # Fetch discovered applications for the current user
                records, app_names = fetch_discovered_apps(user_id)

                # Save the results for the current user
                save_results(app_names, records, user_id)

                logger.info(f"Process completed for user {user_id}. Total records: {len(records)}")
                logger.info(f"Unique applications found for user {user_id}: {len(app_names)}")

            # Update the start index for the next batch
            start_index += len(batch_user_ids)

            # If there are more user IDs to process, wait before starting the next batch
            if start_index < total_user_ids:
                logger.info(
                    f"Finished processing batch {start_index // batch_size}. Waiting {delay_seconds} seconds before the next batch...")
                time.sleep(delay_seconds)  # Wait for the specified delay

    except Exception as e:
        logger.error(f"Process failed: {str(e)}")
        raise


if __name__ == "__main__":
    main()  # Entry point of the script
```

### Sample <br>
![image](https://github.com/user-attachments/assets/b5419be7-0636-4a51-87f8-38ff4f4abfaa)

