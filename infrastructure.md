## Setting up the infrastructure using G-Cloud console

## 1.1 Google Cloud Storage Setup via Console

1. *Create a new Cloud Storage bucket:*
   - Navigate to the [Google Cloud Console](https://console.cloud.google.com/)
   - In the left navigation menu, select "Cloud Storage" > "Buckets"
   - Click the "CREATE" button
   - Enter "medical-documents-bucket" (or your preferred name) as the bucket name
   - Select your preferred region (e.g., "us-central1")
   - Keep the default storage class (Standard)
   - Under "Access Control", select "Fine-grained" (recommended for more control)
   - Click "CREATE" to create the bucket

2. *Configure Cloud Function trigger notification:*
   - In your bucket's details view, go to the "CONFIGURATION" tab
   - Scroll down to find "Event notifications" and click "ADD NOTIFICATION"
   - Enter a notification name (e.g., "document-upload")
   - For "Event type", select "Finalize/Create"
   - For "Destination", select "Cloud Function" (you'll need to create the function first or come back to this step later)
   - Select your Cloud Function from the dropdown
   - Click "CREATE" to set up the notification

## 1.2 Document AI Setup via Console

1. *Create a Document AI processor:*
   - Navigate to the [Google Cloud Console](https://console.cloud.google.com/)
   - In the search bar at the top, type "Document AI" and select it from the results
   - Click "ENABLE" if you haven't enabled the Document AI API yet
   - Click "CREATE PROCESSOR"
   - Select the processor type:
     - For general documents, select "Document OCR"
     - For medical documents, select "Form Parser" or a specialized medical processor if available
   - Enter a processor name (e.g., "medical-document-processor")
   - Select your preferred region (e.g., "us")
   - Click "CREATE" to create the processor
   - *Important:* Note the processor ID shown in the processor details page (it will look like a long alphanumeric string) - you'll need this for your Cloud Function

## 1.3 Firebase/Firestore Setup via Console

1. *Create or connect a Firebase project:*
   - Navigate to the [Firebase Console](https://console.firebase.google.com/)
   - Click "Add project"
   - Select your existing Google Cloud project from the dropdown
   - Follow the setup prompts, enabling Google Analytics if desired
   - Click "Create project" to complete the setup

2. *Initialize Firestore database:*
   - In the Firebase Console, select your project
   - In the left navigation menu, click "Firestore Database"
   - Click "Create database"
   - Select "Start in production mode" for more secure default rules
   - Choose your preferred location for the Firestore database
   - Click "Enable" to create the database

3. *The medical_metadata collection will be created automatically* when your Cloud Function first runs, but you can manually create it if desired:
   - In the Firestore Database view, click "Start collection"
   - Enter "medical_metadata" as the Collection ID
   - You can add a test document with some placeholder fields or click "Cancel" as the Cloud Function will populate this collection automatically

## Setting Up Cloud Function via Console

1. *Create a new Cloud Function:*
   - Navigate to the [Google Cloud Console](https://console.cloud.google.com/)
   - In the left navigation menu, select "Cloud Functions"
   - Click "CREATE FUNCTION"
   - Configure the basics:
     - Function name: "process-file-event"
     - Region: Select your preferred region (e.g., "us-central1")
     - Trigger type: "Cloud Storage"
     - Event type: "Finalize/Create"
     - Bucket: Select your "medical-documents-bucket" from the dropdown
   - Click "SAVE" to continue
   - Configure the runtime:
     - Runtime: Select "Python 3.10"
     - Entry point: "process_file_event"
     - In the inline editor, paste the Cloud Function code (the full Python code provided in the documentation)
     - Click "requirements.txt" and paste the following dependencies:
       
       firebase-admin>=5.0.0 <br/>
       google-cloud-storage>=2.0.0 <br/>
       google-cloud-documentai>=2.0.0 <br/>
       google-cloud-dlp>=3.0.0 <br/>
       functions-framework>=3.0.0
       
   - Click "main.py" and paste the follwing:
   ```bash
   import firebase_admin
   from firebase_admin import credentials, firestore
   from google.cloud import storage, documentai, dlp_v2
   import uuid
   import os
   import functions_framework
   import random  # For demonstration purposes

   # Initialize Firestore
   cred = credentials.ApplicationDefault()
   firebase_admin.initialize_app(cred)
   db = firestore.client()

   # Set Google Cloud Configurations
   PROJECT_ID = "mock-project-454115"  # Replace with your actual project ID
   PROCESSOR_ID = "e2878488ba80e60e"  # Replace with your Document AI Processor ID
   LOCATION = "us"  # Match Document AI region

   @functions_framework.cloud_event
   def process_file_event(cloud_event):
       """Handles file uploads in Cloud Storage."""
       
       data = cloud_event.data
       event_type = cloud_event["type"]  # Checks if it's an upload event
       bucket_name = data['bucket']
       file_name = data['name']

       print(f"ðŸ“¢ Event Received: {event_type} for {file_name}")  # Debug log

       if event_type == "google.cloud.storage.object.v1.finalized":
           process_file_upload(bucket_name, file_name)

   def process_file_upload(bucket_name, file_name):
       """Processes file uploaded to Cloud Storage, extracts text, and stores metadata in Firestore."""
       
       # First, check if a document with this file_name already exists
       existing_docs = db.collection("medical_metadata").where("file_name", "==", file_name).limit(1).stream()
       existing_doc = None
       
       for doc in existing_docs:
           existing_doc = doc
           print(f"ðŸ“Œ Found existing document for {file_name} with ID: {doc.id}")
           break
       
       # If document exists, use its ID, otherwise generate a new one
       if existing_doc:
           doc_id = existing_doc.id
           print(f"ðŸ”„ Updating existing document with ID: {doc_id}")
       else:
           # Generate a unique ID
           unique_id = str(uuid.uuid4())
           
           # Create a Firestore document ID with the format: <file_name>_<unique_id>
           # First, get the base filename without path
           base_file_name = os.path.basename(file_name)
           # Remove any special characters that might be problematic in Firestore IDs
           safe_file_name = ''.join(c if c.isalnum() or c in ['-', '_'] else '_' for c in base_file_name)
           # Create the document ID
           doc_id = f"{safe_file_name}_{unique_id}"
           print(f"ðŸ†• Creating new document with ID: {doc_id}")

       # Check file type before processing
       file_extension = os.path.splitext(file_name)[1].lower()
       mime_type = "application/pdf"  # Default
       
       if file_extension == ".pdf":
           mime_type = "application/pdf"
       elif file_extension in ['.jpg', '.jpeg']:
           mime_type = "image/jpeg"
       elif file_extension == '.png':
           mime_type = "image/png"
       elif file_extension == '.tiff':
           mime_type = "image/tiff"
       else:
           print(f"âš ï¸ Warning: Unsupported file type: {file_extension}. Attempting to process as PDF.")

       try:
           # Process the file using Document AI
           extracted_text, department = analyze_document(bucket_name, file_name, mime_type)

           # Mask PII Data using DLP
           masked_text = mask_pii_data(extracted_text)
           
           # Define security-related attributes
           # In a real system, these would be determined based on document content or external systems
           # For demonstration, we're using random/fixed values
           clearance_levels = ["low", "medium", "high", "top_secret"]
           clearance_level = random.choice(clearance_levels)
           
           # Store metadata and AI-extracted text in Firestore with security attributes
           doc_ref = db.collection("medical_metadata").document(doc_id)
           doc_ref.set({
               # Basic metadata
               "file_name": file_name,
               "bucket_name": bucket_name,
               "extracted_text": masked_text,
               "status": "AI Processed",
               "processing_date": firestore.SERVER_TIMESTAMP,
               "last_updated": firestore.SERVER_TIMESTAMP,
               
               # ABAC-related attributes
               "department": department,
               "clearance_level": clearance_level,
               "can_access_research": random.choice([True, False]),
               "restricted_access": random.choice([True, False]),
               "patient_consented": random.choice([True, False])
           }, merge=True)  # Using merge=True to update only specified fields if document exists

           print(f"âœ… Metadata and AI results stored for {file_name} with ID: {doc_id}")
       
       except Exception as e:
           # Log the error and store information about the failed processing
           error_message = str(e)
           print(f"âŒ Error processing {file_name}: {error_message}")
           
           # Store error information in Firestore
           doc_ref = db.collection("medical_metadata").document(doc_id)
           doc_ref.set({
               "file_name": file_name,
               "bucket_name": bucket_name,
               "status": "Processing Failed",
               "error": error_message,
               "processing_date": firestore.SERVER_TIMESTAMP,
               "last_updated": firestore.SERVER_TIMESTAMP
           }, merge=True)  # Using merge=True to update only specified fields if document exists

   def analyze_document(bucket_name, file_name, mime_type):
       """Downloads the file from Cloud Storage and sends it to Document AI for processing."""
       
       # Initialize Cloud Storage client
       storage_client = storage.Client()
       bucket = storage_client.bucket(bucket_name)
       blob = bucket.blob(file_name)
       
       # Download the file as bytes
       file_content = blob.download_as_bytes()

       # Initialize Document AI client
       client = documentai.DocumentProcessorServiceClient()

       # Set up the processor name
       name = f"projects/{PROJECT_ID}/locations/{LOCATION}/processors/{PROCESSOR_ID}"

       # Configure the request to Document AI
       request = documentai.ProcessRequest(
           name=name,
           raw_document=documentai.RawDocument(
               content=file_content,
               mime_type=mime_type
           )
       )
       
       # Process the document
       result = client.process_document(request=request)
       
       # Extract text from the document
       extracted_text = result.document.text if result.document.text else "No text extracted"
       
       # Extract structured information including potential department information
       document = result.document
       department = "Unknown"
       
       # Look for doctor specialties/departments in the document
       # First, check if Document AI entities contain this information
       for entity in document.entities:
           if entity.type_ == "DEPARTMENT" or "DEPARTMENT" in entity.type_ or entity.type_ == "SPECIALTY" or "SPECIALTY" in entity.type_:
               department = entity.mention_text
               break
       
       # If not found in entities, look for the doctor specialty pattern:
       # "Dr. [Name]" followed by specialty on next line
       if department == "Unknown":
           text_lines = extracted_text.split('\n')
           doctor_line_indices = []
           
           # Find all lines containing "Dr." references
           for i, line in enumerate(text_lines):
               if "Dr. " in line or "Dr " in line or "Doctor " in line:
                   doctor_line_indices.append(i)
           
           # Check the lines right after doctor mentions for specialty/department
           for idx in doctor_line_indices:
               if idx + 1 < len(text_lines):
                   next_line = text_lines[idx + 1].strip()
                   # Check if the next line is a single word or short phrase (likely a specialty)
                   if next_line and len(next_line.split()) <= 3 and not next_line.startswith("Dr"):
                       department = next_line
                       break
           
           # If still unknown, use some common departments for demonstration
           if department == "Unknown":
               departments = ["Cardiology", "Neurology", "Oncology", "Radiology", "Pediatrics", 
                             "Orthopedics", "Dermatology", "Psychiatry", "Emergency Medicine"]
               department = random.choice(departments)
       
       return extracted_text, department

   def mask_pii_data(text):
       """Uses Cloud DLP to mask sensitive PII data in extracted text."""
       
       # If text is empty or too short, return it as is
       if not text or len(text) < 10:
           return text
       
       dlp_client = dlp_v2.DlpServiceClient()
       parent = f"projects/{PROJECT_ID}"

       # Configure DLP to find PII data
       inspect_config = {
           "info_types": [
               {"name": "EMAIL_ADDRESS"}, 
               {"name": "PHONE_NUMBER"}, 
               {"name": "PERSON_NAME"},
               {"name": "US_SOCIAL_SECURITY_NUMBER"},
               {"name": "CREDIT_CARD_NUMBER"},
               {"name": "DATE_OF_BIRTH"}
           ],
           "min_likelihood": dlp_v2.Likelihood.POSSIBLE,
           "include_quote": False
       }

       # Masking configuration
       deidentify_config = {
           "info_type_transformations": {
               "transformations": [{
                   "primitive_transformation": {
                       "replace_config": {"new_value": {"string_value": "[REDACTED]"}}
                   }
               }]
           }
       }

       # Create DLP request
       item = {"value": text}
       request = {
           "parent": parent,
           "inspect_config": inspect_config,
           "item": item,
           "deidentify_config": deidentify_config,
       }

       # Perform PII detection and masking
       response = dlp_client.deidentify_content(request=request)
       masked_text = response.item.value

       return masked_text
   ```
    - Click on Deploy.
   