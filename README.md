# Amino-Acid-Properties-Prediction

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


## DATABASE SETUP


## DEFINE THE FUNCTION TOOL FOR THE LLM`


## INITIALIZE THE LLM WITH THE TOOL


## MAIN INTERACTION LOOP


