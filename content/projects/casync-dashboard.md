---
title: Dashboard For Casync
date: 2018-12-08
images: []
tags: ["rustlang", "web-assembly"]
githubUrl: "https://github.com/krishnakumar4a4/casync-dashboard"
summary: "A dashboard(in web assembly) for managing content for casync,desync and casync-rs type tools"
---
A frontend dashboard(in web assembly) for managing content for casync,desync and casync-rs type tools 

## Build and run
``cargo web start --target wasm32-unknown-unknown``

## Screenshots of dashboard

### View for uploading chunks, indexes and blobs

{{< figure src="https://github.com/krishnakumar4a4/casync-dashboard/blob/master/screenshots/upload_view.png?raw=true">}}

### View for showing all chunks

{{< figure src="https://github.com/krishnakumar4a4/casync-dashboard/blob/master/screenshots/chunks_view.png?raw=true">}}

### View for showing all indexes

{{< figure src="https://github.com/krishnakumar4a4/casync-dashboard/blob/master/screenshots/indexes_view.png?raw=true">}}

### View for showing all tags

{{< figure src="https://github.com/krishnakumar4a4/casync-dashboard/blob/master/screenshots/tags_view.png?raw=true">}}

### View for showing download trends graph of chunks

{{< figure src="https://github.com/krishnakumar4a4/casync-dashboard/blob/master/screenshots/chunks_graph_view.png?raw=true">}}

### View for showing download trends graph of indexes

{{< figure src="https://github.com/krishnakumar4a4/casync-dashboard/blob/master/screenshots/indexes_graph_view.png?raw=true">}}

## How does this work?
### Frontend
- Current project is the frontend which gives lot of features like
  - Look at all chunks, indexes and their statistics like size, modified time, download count etc
  - Upload a blob, backend will take of chunking it and storing them in right places for serving
  - Upload index and chunks separately, if needed
  - Look at chunks related to an index
  - Attach tags to indexes (WIP)
### [Backend](https://github.com/krishnakumar4a4/casync-dashboard-backend)
- A webserver backed by postgresql database provides APIs as well as stores the metadata related to chunks, indexes etc. 
  This backend has much more flexibility and can also be used independent of the frontend dashboard.
