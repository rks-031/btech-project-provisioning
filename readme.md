## Security in Smart Healthcare Systems

The system implements a secure medical document processing pipeline using Google Cloud services with Attribute-Based Access Control (ABAC) in Firestore.

Key components:
1. Medical documents are uploaded to Cloud Storage
2. Cloud Function is triggered by the upload event
3. Document AI processes the document to extract text and metadata
4. Cloud DLP masks sensitive PII information
5. Processed data is stored in Firestore with security attributes
6. Firestore Security Rules enforce ABAC for different user roles