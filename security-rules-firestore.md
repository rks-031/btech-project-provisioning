## Firestore Security Rules

1. Navigate to Firebase Console ➡️’ Firestore ➡️’ Rules
2. Replace the default rules with our ABAC security rules:

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Function to check if user is authenticated
    function isAuthenticated() {
      return request.auth != null;
    }

    // Function to check if user is admin
    function isAdmin() {
      return isAuthenticated() && request.auth.token.role == 'admin';
    }

    // Function to check if user is a doctor
    function isDoctor() {
      return isAuthenticated() && request.auth.token.role == 'doctor';
    }

    // Function to check if user is a nurse
    function isNurse() {
      return isAuthenticated() && request.auth.token.role == 'nurse';
    }

    // Function to check if user is a researcher
    function isResearcher() {
      return isAuthenticated() && request.auth.token.role == 'researcher';
    }

    // Function to check if emergency access is valid (30-minute expiry)
    function isEmergencyAccessValid(resource) {
      return resource.data.emergency_access == true &&
             request.time < resource.data.emergency_expiry &&
             request.time - resource.data.emergency_granted_time < duration.value(30, "m");
    }

    // Function to check if user is authorized to access a medical document
    function canAccessMedical(resource) {
      return resource.data != null && (
        isAdmin() || 
        (isDoctor() && request.auth.token.department == resource.data.department) ||
        (isNurse() && request.auth.token.is_trainee == false) ||
        (isResearcher() && resource.data.can_access_research == true) ||
        isEmergencyAccessValid(resource)  // Allow temporary emergency access for 30 minutes
      );
    }

    // Rules for medical_metadata collection
    match /medical_metadata/{documentId} {
      // Allow read if user passes access control checks
      allow read: if canAccessMedical(resource);

      // Only admins and doctors can create/update documents
      allow create, update: if isAdmin() || 
                             (isDoctor() && request.resource.data.department == request.auth.token.department);

      // Only admins can delete
      allow delete: if isAdmin();
    }

    // Rules for users collection
    match /users/{userId} {
      // Users can read/write their own documents
      allow read, write: if request.auth.uid == userId;

      // Admins can read/write all user documents
      allow read, write: if isAdmin();
    }
  }
}
```
3. Click "Publish" to apply the rules
