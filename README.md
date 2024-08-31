## Cerner Practitioner App

Description

The following repository outlines a basic practitoner app that utilizes the OAuth2 authentication to access patient data from Cerner. The app displays a patients Vital Signs in a user friendly manner and allows for a practitoners to create a new temperature observation and post it to the FHIR server. The newly created vital sign then appears in the vital sign table.

Features

 OAuth2 Authentication: Signing directly into Cerner and Redirected to App 

 Data Extraction: Patient Banner displaying Gender, Name, DOB and Previous Vital Signs in a table

 User Friendly Display: Extracted data is displayed in tables and boxes
  

Tech Stack

    Frontend: Svelte, TypeScript, FHIR, CSS
