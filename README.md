# FileReach-
Focuses on extending file access via SMS

To solve the problem of creating a system for uploading multiple files with additional metadata (document owner’s name, address, email ID, and telephone number), integrating MongoDB for NoSQL data storage, and sending SMS notifications with a download link, we will separate the project into two distinct solutions: one for the backend (using Python Django) and one for the frontend (using ReactJS). Here is the complete breakdown of the solution:

Key Features:
Multiple File Upload: Users can upload multiple files at once.
Document Owner Metadata: Name, address, email, and telephone number are submitted along with the file(s).
SMS Notification: The telephone number provided will receive an SMS containing the download link for the uploaded files.
Download without Authentication: The generated download link should not require authentication.
MongoDB for NoSQL storage: File metadata will be stored in MongoDB, including a reference to the file paths.
Backend in Django: For handling file uploads, generating the link, and storing metadata in MongoDB.
Frontend in React: For user interaction and file upload UI.
Folder Structure
Backend (Django)
markdown
Copy code
backend/
│
├── manage.py
├── backend/
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   ├── wsgi.py
│   ├── asgi.py
│
├── api/
│   ├── __init__.py
│   ├── models.py
│   ├── views.py
│   ├── serializers.py
│   ├── urls.py
│
├── sms/
│   ├── __init__.py
│   ├── send_sms.py
│
├── uploads/
│   └── media/
│
├── requirements.txt
└── Dockerfile
Frontend (React)
java

frontend/
│
├── public/
│   └── index.html
│
├── src/
│   ├── components/
│   │   ├── FileUpload.js
│   │   ├── UploadForm.js
│   ├── App.js
│   ├── index.js
│   ├── api.js
│
├── package.json
├── Dockerfile
└── .env
Backend (Django) Setup
Step 1: Install Required Packages
bash

pip install django djangorestframework pymongo twilio django-cors-headers
Step 2: Backend Configuration
1. settings.py

Integrate MongoDB using djongo or pymongo.
Enable CORS for frontend communication.
python

# settings.py

INSTALLED_APPS = [
    ...
    'rest_framework',
    'corsheaders',
    'api',
]

DATABASES = {
    'default': {
        'ENGINE': 'djongo',
        'NAME': 'your_database_name',
    }
}

CORS_ALLOW_ALL_ORIGINS = True

MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'uploads/media')
2. models.py

python
Copy code
from django.db import models

class Document(models.Model):
    owner_name = models.CharField(max_length=255)
    address = models.TextField()
    email = models.EmailField()
    phone_number = models.CharField(max_length=15)
    file_path = models.TextField()
    download_url = models.URLField()

    def __str__(self):
        return self.owner_name
3. serializers.py

python
Copy code
from rest_framework import serializers
from .models import Document

class DocumentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Document
        fields = '__all__'
4. views.py

python
Copy code
from rest_framework.response import Response
from rest_framework.views import APIView
from rest_framework.parsers import MultiPartParser, FormParser
from .models import Document
from .serializers import DocumentSerializer
from .send_sms import send_sms
import os

class FileUploadView(APIView):
    parser_classes = (MultiPartParser, FormParser)

    def post(self, request, *args, **kwargs):
        files = request.FILES.getlist('files')
        owner_name = request.data.get('owner_name')
        address = request.data.get('address')
        email = request.data.get('email')
        phone_number = request.data.get('phone_number')

        file_paths = []
        for file in files:
            file_path = os.path.join('uploads', file.name)
            with open(file_path, 'wb+') as destination:
                for chunk in file.chunks():
                    destination.write(chunk)
            file_paths.append(file_path)

        # Store file metadata in MongoDB
        document = Document.objects.create(
            owner_name=owner_name,
            address=address,
            email=email,
            phone_number=phone_number,
            file_path=", ".join(file_paths),
            download_url=f"http://localhost:8000/media/{file.name}"
        )
        document.save()

        # Send SMS notification with download link
        send_sms(phone_number, document.download_url)

        return Response({"message": "Files uploaded successfully."})
5. send_sms.py

python

from twilio.rest import Client

def send_sms(phone_number, download_url):
    account_sid = 'your_account_sid'
    auth_token = 'your_auth_token'
    client = Client(account_sid, auth_token)

    message = client.messages.create(
        body=f"Your file is available for download at: {download_url}",
        from_='+1234567890',
        to=phone_number
    )
6. urls.py

python

from django.urls import path
from .views import FileUploadView

urlpatterns = [
    path('upload/', FileUploadView.as_view(), name='file-upload'),
]
Step 3: Setting up MongoDB
Install and configure MongoDB, then start the MongoDB service locally or use a cloud-based MongoDB provider.
Step 4: Run Django Backend
bash

python manage.py makemigrations
python manage.py migrate
python manage.py runserver
Frontend (React) Setup
Step 1: Install Required Packages
bash

npx create-react-app frontend
cd frontend
npm install axios
Step 2: Frontend Code
1. api.js

javascript

import axios from 'axios';

export const uploadFiles = async (formData) => {
    try {
        const response = await axios.post('http://localhost:8000/upload/', formData, {
            headers: {
                'Content-Type': 'multipart/form-data'
            }
        });
        return response.data;
    } catch (error) {
        console.error("There was an error uploading the files!", error);
    }
};
2. UploadForm.js

javascript

import React, { useState } from 'react';
import { uploadFiles } from '../api';

const UploadForm = () => {
    const [formData, setFormData] = useState({
        owner_name: '',
        address: '',
        email: '',
        phone_number: '',
        files: []
    });

    const handleFileChange = (e) => {
        setFormData({ ...formData, files: e.target.files });
    };

    const handleSubmit = async (e) => {
        e.preventDefault();
        const form = new FormData();
        form.append('owner_name', formData.owner_name);
        form.append('address', formData.address);
        form.append('email', formData.email);
        form.append('phone_number', formData.phone_number);
        for (let file of formData.files) {
            form.append('files', file);
        }

        const result = await uploadFiles(form);
        console.log(result);
    };

    return (
        <form onSubmit={handleSubmit}>
            <input type="text" placeholder="Owner Name" onChange={e => setFormData({ ...formData, owner_name: e.target.value })} />
            <input type="text" placeholder="Address" onChange={e => setFormData({ ...formData, address: e.target.value })} />
            <input type="email" placeholder="Email" onChange={e => setFormData({ ...formData, email: e.target.value })} />
            <input type="text" placeholder="Phone Number" onChange={e => setFormData({ ...formData, phone_number: e.target.value })} />
            <input type="file" multiple onChange={handleFileChange} />
            <button type="submit">Upload Files</button>
        </form>
    );
};

export default UploadForm;
3. App.js

javascript
Copy code
import React from 'react';
import UploadForm from './components/UploadForm';

function App() {
  return (
    <div className="App">
      <h1>File Upload</h1>
      <UploadForm />
    </div>
  );
}

export default App;
Step 3: Run the React Frontend
bash

npm start
Advanced Enhancements:
File Expiry: Add an expiry time for the download links, after which they will be invalid.
File Encryption: Ensure secure file storage with encryption.
Progress Bar: Implement a progress bar for large file uploads.
User-Friendly SMS: Include a more detailed message in the SMS, like


Here are a few name suggestions for your solution that involve uploading files, handling document metadata, and sending SMS notifications for secure access:

FileLinker – Highlights the linking aspect between the file upload and the SMS notification with the download link.
DocuSMS – Combines document management and SMS notification features.
QuickFileShare – Emphasizes fast and secure file sharing via SMS.
FilePath – Focuses on the direct path/link for files without authentication.
SnapShare – Suggests a quick, seamless file-sharing experience.
MetaSend – Highlights metadata collection and sending the access link.
SwiftDocs – Emphasizes quick document upload and access.
LinkDocs – Focuses on sharing files via download links.
DocuFlow – Reflects a smooth flow from file upload to download link delivery.
FileReach – Focuses on extending file access via SMS.

