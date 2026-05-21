## Overview

A webapp for viewing and recording St. Paul crime data. Written primarily in VueJs, using Nominatim, Leaflet, and St. Paul crime APIs. Made for CISC 375 - Web Development at the University of St. Thomas. Developed by Ryan Soltis.

## Usage Instructions

After successfully getting the webapp running and going to the local URL (after following the Installation Instructions, should be http://localhost:5173), and entering the URL the API is being hosted on, you are able to:
- 

## Installation Instructions

### Prerequisites
- Node.js 18+

### 1. Setup Crime API [(Link to repo if want to clone on web)](https://github.com/ryansoltz/CISC375-Project3)
```bash
git clone https://github.com/ryansoltz/CISC375-Project3
cd CISC375-Project3
#Copy crime database to db folder
npm install
node rest_server.mjs
#The url to the API should be http://localhost:8000 (important for step 2)
```
### 2. Run the Webapp
```bash
#Can use this code or can clone using your IDE
git clone https://github.com/ryansoltz/CISC375-Project4
cd CISC375-Project4
npm install
npm run dev
```

