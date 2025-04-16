# Amino-Acid-Properties-Prediction Using Gen AI

## INSTALLTION AND IMPORT
```python
# Remove conflicting packages 
# !pip uninstall -qqy jupyterlab

# Installing necessary libraries
!pip install -U -q pandas "google-generativeai>=0.4.0" # Use a version supporting function calling

import google.generativeai as genai
import pandas as pd
import sqlite3
import json
import os # For API key management (recommended)
import typing # For function type hints

print("installation and importing DONE!!!")
```

## CONFIGURATION
```python
GOOGLE_API_KEY = "AIzaSyBZw0gtxf3AtS-8ieqTPNxBSbe8-8Qnaiw" # <--- REPLACE THIS with your API

# Configure the GenAI client
genai.configure(api_key=GOOGLE_API_KEY)
print("Google GenAI client configuration DONE.")

# TAking the csv file and converting into db
csv_file = "/kaggle/input/aminoacids-physical-and-chemical-properties/aminoacids.csv"
db_file = "/kaggle/working/aminoacids.db"
```

## DATABASE SETUP
```python
def setup_database(csv_path: str, db_path: str) -> bool:
    """Loads amino acid data from CSV into an SQLite database."""
    print(f"Attempting database setup: {csv_path} -> {db_path}")
        # Load CSV data using pandas
    df = pd.read_csv(csv_path)
    print(f"Successfully loaded CSV. Columns: {df.columns.tolist()}")

        # Connect to SQLite database (creates the file if it doesn't exist)
    conn = sqlite3.connect(db_path)

        # Write the pandas DataFrame to an SQL table named 'amino_acids'
        # if_exists='replace' will overwrite the table if it already exists
    df.to_sql("amino_acids", conn, if_exists="replace", index=False)

        # Commit changes and close connection
    conn.commit()
    conn.close()
print("Database setup successful.")
```

## DEFINE THE FUNCTION TOOL FOR THE LLM`
```python
def get_amino_acid_properties(amino_acid_name: str) -> typing.Dict[str, typing.Any]:
    """
    Retrieves chemical and physical properties for a given amino acid NAME
    from the 'amino_acids' database table.

    Args:
        amino_acid_name: The full name of the amino acid (e.g., 'Alanine', 'Glycine').

    Returns:
        A dictionary containing the properties of the amino acid if found,
        otherwise an error message dictionary.
    """
    print(f"\n[Function Called] Searching database for: '{amino_acid_name}'")
    conn = None # Initialize connection variable
    try:
        # Connect to the database file
        conn = sqlite3.connect(db_file)
        # Use Row factory to access columns by name (like a dictionary)
        conn.row_factory = sqlite3.Row
        cursor = conn.cursor()

        # Execute SQL query safely using parameterization (?)
        # Matching based on the 'Name' column
        cursor.execute("SELECT * FROM amino_acids WHERE Name = ?", (amino_acid_name,))
        result = cursor.fetchone() # Get the first matching row

        if result:
            # Convert the database row into a standard Python dictionary
            properties = dict(result)
            print(f"[Function Result] Found properties for {amino_acid_name}.")
            # Simple data cleaning for pKx3 (optional, but good practice)
            if properties.get('pKx3') is not None:
                try:
                    # Try converting to float if not None
                     properties['pKx3'] = float(properties['pKx3'])
                except (ValueError, TypeError):
                    # If conversion fails (e.g., it's text like "None"), set to None
                    properties['pKx3'] = None
            return properties # Return the dictionary of properties
        else:
            print(f"[Function Result] Amino acid '{amino_acid_name}' not found.")
            # Return an error dictionary if not found
            return {"error": f"Amino acid '{amino_acid_name}' not found in database."}

    except sqlite3.Error as e:
        print(f"[Function Error] Database error: {e}")
        return {"error": f"Database query failed: {e}"}
    except Exception as e:
        print(f"[Function Error] Unexpected error: {e}")
        return {"error": f"An unexpected error occurred: {e}"}
    finally:
        # Ensure the database connection is closed
        if conn:
            conn.close()
            # print("[Function End] Database connection closed.") # Optional debug message

print("Function tool defined.")
```

## INITIALIZE THE LLM WITH THE TOOL
```python
# Choose a model that supports function calling
model = genai.GenerativeModel(
    model_name='gemini-1.5-flash', # Or 'gemini-1.0-pro', 'gemini-1.5-pro' etc.
    # IMPORTANT: Tell the model about the function(s) it can use
    tools=[get_amino_acid_properties]
    )

    # Start a chat session with automatic function calling enabled
    # This means the library will automatically run your function when the LLM requests it
chat = model.start_chat(enable_automatic_function_calling=True)
print("Generative Model initialized successfully.")
```

## MAIN INTERACTION LOOP
```python
# First, setup the database using the function defined in DEFINE THE FUNCTION TOOL FOR THE LLM
if not setup_database(csv_file, db_file):
    print("\nDatabase setup failed. Cannot continue. Please check CSV path and file.")
    exit() # Stop the script if DB setup fails

print("\n--- Amino Acid Property Finder ---")
print("Enter the full name of an amino acid (e.g., Alanine, Histidine).")
print("Type 'quit' or 'exit' to stop.")

while True:
    # Get input from the user
    user_input = input("\nEnter amino acid name: ").strip().capitalize()

    # Check if the user wants to quit
    if user_input.lower() in ['quit', 'exit']:
        print("Exiting program.")
        break

    # Make sure user entered something
    if not user_input:
        continue

    # Create the prompt for the LLM. Ask clearly for properties and JSON output.
    # The LLM will see this and decide to use the 'get_amino_acid_properties' tool.
    prompt = f"Please retrieve the chemical and physical properties for the amino acid '{user_input}' from the database tool and provide the result strictly as a JSON object."
    # prompt = f"What are the properties of {user_input}? Use the available tool and give me JSON." # Alternative prompt

    print(f"\nSending request to LLM for: '{user_input}'...")

    try:
        # Send the prompt to the chat session
        # Because automatic function calling is ON, the following happens:
        # 1. LLM sees the prompt and the 'get_amino_acid_properties' tool description.
        # 2. LLM decides to call the tool with `amino_acid_name=user_input`.
        # 3. The `google-genai` library executes your `get_amino_acid_properties` function.
        # 4. Your function queries the DB and returns a dictionary (properties or error).
        # 5. The library sends this dictionary result back to the LLM.
        # 6. The LLM uses the dictionary result to formulate the final text response.
        response = chat.send_message(prompt)

        print("\n--- LLM Response ---")
        # The final response text should contain the data, ideally formatted as JSON
        response_text = response.text

        # Try to parse and print as formatted JSON for better readability
        try:
            # Attempt to load the text as JSON
            json_output = json.loads(response_text)
            # Pretty-print the JSON
            print(json.dumps(json_output, indent=2))
        except json.JSONDecodeError:
            # If it's not valid JSON (e.g., an error message), print the raw text
            print("Response was not valid JSON, printing raw text:")
            print(response_text)
        except AttributeError:
            print("Error: Could not access response text.")
            # print("Full response object:", response) # For debugging

    except Exception as e:
        print(f"\nAn ERROR occurred during the LLM interaction: {e}")
        print("Please check your API key, internet connection, and input.")

print("\n--- Interaction loop finished. ---")
```

