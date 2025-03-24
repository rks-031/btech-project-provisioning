### 4.1 Testing the Document Processing

1. Upload a sample medical document to your Cloud Storage bucket:
   ```bash
   gsutil cp sample_medical_report.pdf gs://medical-documents-bucket/
   ```

2. Check Cloud Functions logs to verify processing:
   ```bash
   gcloud functions logs read process_file_event
   ```

3. Navigate to Firestore in Firebase Console to confirm the document was processed and stored with appropriate metadata

### 4.2 Testing Security Rules in Firebase Rules Playground

1. Navigate to Firebase Console ➡️’ Firestore ➡️’ Rules ➡️’ Rules Playground

2. Set up the following test scenarios:

#### Scenario 1: Admin Access Test
- **Operation**: get
- **Path**: `/databases/(default)/documents/medical_metadata/{docId}`
- **Authentication**: Admin User
  ```json
  {
    "uid": "admin-user-123",
    "token": {
      "sub": "admin-user-123",
      "aud": "mock-project-454115",
      "role": "admin",
      "email": "admin@hospital.com",
      "firebase": {
        "sign_in_provider": "custom"
      }
    }
  }
  ```
- **Document Data**:
  ```json
  {
    "file_name": "cardiac_report.pdf",
    "bucket_name": "medical-documents-bucket",
    "department": "Cardiology",
    "clearance_level": "medium",
    "can_access_research": true,
    "status": "AI Processed"
  }
  ```
- **Expected Result**: ALLOW

#### Scenario 2: Doctor Access Test (Same Department)
- **Operation**: get
- **Path**: `/databases/(default)/documents/medical_metadata/{docId}`
- **Authentication**: Cardiologist 
  ```json
  {
    "uid": "doctor-user-456",
    "token": {
      "sub": "doctor-user-456",
      "aud": "mock-project-454115",
      "role": "doctor",
      "department": "Cardiology",
      "email": "cardiologist@hospital.com",
      "firebase": {
        "sign_in_provider": "custom"
      }
    }
  }
  ```
- **Document Data**: Same as above (Cardiology)
- **Expected Result**: ALLOW

#### Scenario 3: Doctor Access Test (Different Department)
- **Operation**: get
- **Path**: `/databases/(default)/documents/medical_metadata/{docId}`
- **Authentication**: Cardiologist (same as above)
- **Document Data**:
  ```json
  {
    "file_name": "brain_scan.pdf",
    "bucket_name": "medical-documents-bucket",
    "department": "Neurology",
    "clearance_level": "high",
    "can_access_research": false,
    "status": "AI Processed"
  }
  ```
- **Expected Result**: DENY

#### Scenario 4: Nurse Access Test (Regular)
- **Operation**: get
- **Path**: `/databases/(default)/documents/medical_metadata/{docId}`
- **Authentication**: Regular Nurse
  ```json
  {
    "uid": "nurse-user-101",
    "token": {
      "sub": "nurse-user-101",
      "aud": "mock-project-454115",
      "role": "nurse",
      "is_trainee": false,
      "email": "nurse@hospital.com",
      "firebase": {
        "sign_in_provider": "custom"
      }
    }
  }
  ```
- **Document Data**:
  ```json
  {
    "file_name": "pediatric_report.pdf",
    "bucket_name": "medical-documents-bucket",
    "department": "Pediatrics",
    "clearance_level": "low",
    "can_access_research": true,
    "status": "AI Processed"
  }
  ```
- **Expected Result**: ALLOW

#### Scenario 5: Nurse Access Test (Trainee)
- **Operation**: get
- **Path**: `/databases/(default)/documents/medical_metadata/{docId}`
- **Authentication**: Trainee Nurse
  ```json
  {
    "uid": "nurse-user-202",
    "token": {
      "sub": "nurse-user-202",
      "aud": "mock-project-454115",
      "role": "nurse",
      "is_trainee": true,
      "email": "trainee@hospital.com",
      "firebase": {
        "sign_in_provider": "custom"
      }
    }
  }
  ```
- **Document Data**: Same as above (Pediatrics)
- **Expected Result**: DENY

#### Scenario 6: Researcher Access Test (Research Allowed)
- **Operation**: get
- **Path**: `/databases/(default)/documents/medical_metadata/{docId}`
- **Authentication**: Researcher
  ```json
  {
    "uid": "researcher-user-303",
    "token": {
      "sub": "researcher-user-303",
      "aud": "mock-project-454115",
      "role": "researcher",
      "email": "researcher@university.edu",
      "firebase": {
        "sign_in_provider": "custom"
      }
    }
  }
  ```
- **Document Data**: Same as Scenario 1 (Cardiology with can_access_research: true)
- **Expected Result**: ALLOW

#### Scenario 7: Researcher Access Test (Research Not Allowed)
- **Operation**: get
- **Path**: `/databases/(default)/documents/medical_metadata/{docId}`
- **Authentication**: Researcher (same as above)
- **Document Data**: Same as Scenario 3 (Neurology with can_access_research: false)
- **Expected Result**: DENY

### Scenario 8: Emergency Access Test (Any Doctor)
- **Operation**: get
- **Path**: `/databases/(default)/documents/medical_metadata/{docId}`
- **Authentication**: Any Doctor
  ```json
  {
  "uid": "doctor-user-789",
  "token": {
    "sub": "doctor-user-789",
    "aud": "mock-project-454115",
    "role": "doctor",
    "department": "General Medicine",
    "email": "doctor@hospital.com",
    "firebase": {
      "sign_in_provider": "custom"
       }
     }
    }
  ```
- **Document Data**:
  ```json
  {
  "file_name": "brain_scan.pdf",
  "bucket_name": "medical-documents-bucket",
  "department": "Neurology",
  "clearance_level": "high",
  "can_access_research": false,
  "status": "AI Processed",
  "emergency_access": true,
  "emergency_granted_time": "2025-03-16T10:00:00Z",
  "emergency_expiry": "2025-03-16T10:30:00Z"
   }
  ```
- **Expected Result**: ALLOW (Only within 30 minutes of emergency_granted_time)

3. Run each scenario and verify the expected results
