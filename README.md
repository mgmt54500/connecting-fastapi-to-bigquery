# Logging API Data to BigQuery

This guide walks you through connecting your FastAPI Calculator API to **Google Cloud BigQuery**. You will learn how to create a database, set up secure permissions, and update your code to log every calculation.

## Overview

1. [Select Your Google Cloud Project](#step-1-select-your-google-cloud-project)
2. [Create a Dedicated Service Account](#step-2-create-a-dedicated-service-account)
3. [Add the Service Account to Cloud Run](#step-3-add-the-service-account-to-cloud-run)
4. [Create the BigQuery Table](#step-4-create-the-bigquery-table)
5. [Update Your Application to Add BigQuery Dependencies](#step-5-update-your-application-to-add-bigquery-dependencies)
6. [Update the API Code](#step-6-update-the-api-code)
7. [Test Locally](#step-7-test-locally)
8. [Push to GitHub and Test Remotely](#step-8-push-to-github-and-test-remotely)

---
## Step 1: Select Your Google Cloud Project

1. In the [Google Cloud Console](https://console.cloud.google.com), ensure the project that contains your Calculator API on Cloud Run is selected

---

## Step 2: Create a Dedicated Service Account

For security, we create a specific "identity" for your API.

1. Navigate to **IAM & Admin > Service Accounts**.
2. Click **+ Create Service Account**.
3. **Name**: `fastapi-bq-accessor`
4. **Grant Access**: Search for and select these two roles:
    * **BigQuery Data Editor** (to write rows).
    * **BigQuery Job User** (to run the insert task).

5. Click **Done**.

---

## Step 3: Add the Service Account to Cloud Run

We need to tell our Cloud Run service which Service Account to use when making a connection to BigQuery.

1. Search for Cloud Run
2. Choose **Services**
3. Choose your **calculator-api** service
4. Click the **Revisions** tab
5. Choose the latest revision
6. On the panel that opens to the right, choose the **Security** tab
7. Click the name of the existing (default) security account
8. On the new page, type `fastapi-bq-accessor` for the Service Account name and click Save
9. Enter a description and click Save

---

## Step 4: Create the BigQuery Table

Create a new dataset and table using the BigQuery interface.

1. Search for **BigQuery** in the top search bar
2. In the **Explorer** pane on the left, click the three dots (`⋮`) next to your Project ID and select **Create dataset**
    * **Dataset ID**: `calculator`
    * **Location type**: us-central1
    * Click **Create Dataset**

3. At the top of the BigQuery interface, choose the **Untitled query** tab
4. Copy the following query and paste it into the tab

> [!IMPORTANT]
> You MUST update your Google Cloud project name in the code below for this to work!

```sql
CREATE TABLE `YOUR_PROJECT.calculator.api_logs` (
  request_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
  endpoint STRING,
  result STRING,
  status_code INT64
);
```

---

## Step 5: Update Your Application to Add BigQuery Dependencies

You have completed the necessary updates to the Google Cloud project. Now you must update your application to take advantage of the changes.

1. Open [**Google Cloud Shell**](https://shell.cloud.google.com)
2. In the terminal, navigate to the directory that holds your `calculator-api`
    * EXAMPLE:
    ```bash
    cd calculator-api
    ```
3. At the command line, enter the following command:
    ```bash
    poetry add google-cloud-bigquery
    ```


*This updates your `pyproject.toml` and `poetry.lock` files so Cloud Run knows to install the BigQuery library.*

---

## Step 6: Update the API Code

Open `main.py` in the Cloud Shell Editor.

1. Update the imports at the top of the file to the following

    ```python
    from fastapi import FastAPI, status, HTTPException, Depends
    from google.cloud import bigquery
    ```

2. Add a dependency function to provide a BigQuery client

    ```python
    # Dependency method to provide a BigQuery client
    # This will be used by the other endpoints where a database connection is necessary
    def get_bq_client():
        # client automatically uses Cloud Run's service account credentials
        client = bigquery.Client()
        try:
            yield client
        finally:
            client.close()
    ```

3. AT THE END OF YOUR FILE, add a new endpoint to your API by copy and pasting the following:

> [!IMPORTANT]
> You MUST update your Google Cloud project name in the code below for this to work!


```python
@app.get("/dbwritetest", status_code=200)
def dbwritetest(bq: bigquery.Client = Depends(get_bq_client)):
    """
    Writes a simple test row to a BigQuery table.

    Uses the `get_bq_client` dependency method to establish a connection to BigQuery.
    """
    # Define a Python list of objects that will become rows in the database table
    # In this instance, there is only a single object in the list
    row_to_insert = [
        {
            "endpoint": "/dbwritetest",
            "result": "Success",
            "status_code": 200
        }
    ]
    
    # Use the BigQuery interface to write our data to the table
    # If there are errors, store them in a list called `errors`
    # YOU MUST UPDATE YOUR PROJECT AND DATASET NAME BELOW BEFORE THIS WILL WORK!!!
    errors = bq.insert_rows_json("YOUR-PROJECT.calculator.api_logs", row_to_insert)

    # If there were any errors, raise an HTTPException to inform the user
    if errors:
        # Log the full error to your Cloud Run logs for debugging
        print(f"BigQuery Insert Errors: {errors}")
        
        # Raise an exception to the API user
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail={
                "message": "Failed to log data to BigQuery",
                "errors": errors  # Optional: return specific BQ error details
            }
        )

    # If there were NOT any errors, send a friendly response message to the API caller
    return {"message": "Log entry created successfully"}
```

---

## Step 7: Test Locally

1. Input the following command on the command line to start your API locally

```bash
poetry run uvicorn main:app --reload --port 8080
```

2. Use the Web Preview to test your `dbwritetest` endpoint

3. In another tab, open BigQuery and see if your row was correctly written to your table

4. Press Ctrl+C in the Cloud Shell terminal to stop your server

---

## Step 8: Push to GitHub and Test Remotely

1. Create a new git commit
2. Push your changes to GitHub
3. Wait for Cloud Build and Cloud Run to deploy the new version of your application
4. Test the `dbwritetest` endpoint using the "live" Cloud Run URL
5. Review your table in BigQuery to see if a new row was written