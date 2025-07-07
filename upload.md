Large File Upload System Design (On-Premise)
This document outlines a robust design for handling large file uploads (up to 12GB) using an Angular frontend and a Spring Boot backend, specifically tailored for an on-premise environment without relying on cloud services. The core strategy revolves around file chunking and resumable uploads to ensure reliability and a good user experience.

1. Core Challenges with Large File Uploads
Uploading extremely large files directly in a single request can lead to several issues:

Network Timeouts: Long-running requests are prone to timeouts.

Memory Consumption: Holding the entire file in memory on either the client or server side is impractical and inefficient.

Reliability: Network interruptions can cause the entire upload to fail, requiring a complete restart.

Progress Tracking: Difficult to provide granular progress feedback to the user.

Scalability: Hard to scale a system that processes large single requests.

2. Solution Overview: Chunking and Resumable Uploads
The proposed solution addresses these challenges by breaking the large file into smaller, manageable chunks on the client-side. These chunks are then uploaded individually to the server. The server receives and reassembles these chunks. This approach enables:

Resumability: If an upload fails, only the missing chunks need to be re-uploaded.

Progress Tracking: Each chunk upload can update the overall progress.

Reduced Memory Footprint: Neither client nor server needs to hold the entire file in memory at once.

Improved Reliability: Smaller transfers are less prone to network issues.

3. Frontend Design (Angular)
The Angular frontend will be responsible for selecting the file, splitting it into chunks, managing the upload process for each chunk, tracking progress, and handling retries for failed chunks.

3.1. File Selection and Initial Setup
HTML Input: Use a standard <input type="file"> element.

File Object: Once a file is selected, obtain the File object from the event.

Unique Upload ID: Generate a unique identifier (e.g., a UUID) for each upload session. This ID will be sent with every chunk and used by the backend to identify and reassemble the correct file.

Chunk Size: Define a manageable chunk size (e.g., 5MB to 20MB). This can be configured. A smaller chunk size means more requests but faster recovery from failures; a larger size means fewer requests but potentially larger re-uploads on failure.

3.2. File Chunking
File.slice() API: Use the File.slice() method to extract byte ranges (chunks) from the selected file.

Iterative Processing: Loop through the file, creating Blob objects for each chunk.

Metadata per Chunk: For each chunk, send the following metadata along with the chunk data:

uploadId: Unique identifier for the entire upload.

fileName: Original file name.

fileSize: Total size of the file.

chunkNumber: The sequential index of the current chunk (0-indexed).

totalChunks: Total number of chunks for the file.

chunkSize: The size of the current chunk (important for the last chunk, which might be smaller).

3.3. Upload Process and Resumability
Queue Management: Implement a queue or a controlled concurrent upload mechanism (e.g., RxJS operators like mergeMap with a concurrent parameter) to upload chunks. Avoid uploading all chunks simultaneously to prevent overwhelming the network or server.

Progress Tracking:

Maintain a state for each chunk (e.g., pending, uploading, completed, failed).

Update a global progress bar based on the number of completed chunks or the sum of bytes uploaded.

Resumable Logic:

Initial Check: Before starting an upload, send a request to the backend with the uploadId (if resuming) or fileName and fileSize (if new).

Backend Response: The backend responds with a list of already received chunks for that uploadId.

Client Action: The frontend then only uploads the chunks that are not on the received list.

Error Handling and Retries:

Implement retry logic for failed chunk uploads (e.g., exponential backoff).

If a chunk fails after multiple retries, mark the overall upload as failed and inform the user. Provide an option to manually retry the entire upload from the last known good state.

3.4. UI/UX Considerations
Clear Progress Bar: A visual progress bar showing overall upload completion.

Status Messages: Informative messages (e.g., "Uploading chunk X of Y", "Upload complete", "Upload failed, retrying...").

Pause/Resume Button: Allow users to pause and resume uploads. This involves stopping the chunk upload queue and then re-initiating the resumable logic when resumed.

Cancel Button: Allow users to cancel an upload, which should also trigger a cleanup request to the backend.

File Information: Display the file name, size, and estimated time remaining.

4. Backend Design (Spring Boot)
The Spring Boot backend will handle receiving individual chunks, storing them temporarily, reassembling them, and finally moving the complete file to its permanent storage location.

4.1. API Endpoints
POST /api/upload/initiate:

Request: fileName, fileSize, uploadId (optional, for resuming).

Response: uploadId (if new), receivedChunks (list of chunk numbers already received).

Purpose: Initializes an upload session on the server or checks the status of an existing one.

POST /api/upload/chunk:

Request: uploadId, fileName, fileSize, chunkNumber, totalChunks, chunkSize, file (the chunk data).

Response: 200 OK on success, 500 Internal Server Error on failure.

Purpose: Receives a single file chunk.

POST /api/upload/complete:

Request: uploadId, fileName, fileSize.

Response: 200 OK if file reassembled successfully, 400 Bad Request if chunks are missing.

Purpose: Signals the backend that all chunks have been sent, prompting final assembly and cleanup.

POST /api/upload/cancel:

Request: uploadId.

Response: 200 OK.

Purpose: Cleans up temporary files associated with a cancelled upload.

4.2. Receiving and Storing Chunks
Temporary Storage:

Create a dedicated temporary directory on the server's file system (e.g., /tmp/uploads/{uploadId}/).

Each chunk received will be saved as a separate file within this directory, named by its chunkNumber (e.g., 0.part, 1.part, etc.).

Why temporary storage? This prevents holding large amounts of data in memory and allows for robust recovery if the server restarts during an upload.

MultipartFile: Spring Boot's MultipartFile can be used to receive the chunk data.

Concurrency: Ensure thread-safe operations when writing chunks, especially if multiple chunks for the same uploadId arrive concurrently. Java's NIO or RandomAccessFile can be used for efficient file writing.

Duplicate Chunk Handling: If a chunk is received multiple times (e.g., due to client-side retries), the backend should simply overwrite the existing chunk file.

4.3. Reassembling the File
Trigger: The reassembly process is typically triggered by the POST /api/upload/complete endpoint.

Verification: Before reassembly, verify that all expected chunks are present in the temporary directory for the given uploadId. Compare the number of files with totalChunks and check their sizes.

Concatenation: Read each chunk file sequentially (from 0.part to N.part) and append its contents to a new, final file in the designated permanent storage location.

Error Handling: If chunks are missing or corrupted, return an appropriate error to the client.

4.4. Permanent Storage
Dedicated Storage: Store the reassembled files on a robust on-premise storage solution such as:

Network File System (NFS): For shared storage across multiple backend instances.

Storage Area Network (SAN): High-performance block storage.

Local Disk: If the application runs on a single server and storage capacity is sufficient.

Directory Structure: Organize files logically (e.g., by date, user ID, or a hashed path).

4.5. Cleanup and Lifecycle Management
Successful Upload: After successful reassembly and move to permanent storage, delete the temporary directory and all chunk files associated with the uploadId.

Failed/Cancelled Upload: Implement a mechanism to clean up stale temporary directories and incomplete uploads:

Manual Cleanup: An admin can manually trigger cleanup.

Scheduled Task: A Spring @Scheduled task can periodically scan the temporary directory for uploadId folders that haven't been modified for a certain period (e.g., 24-48 hours) and delete them.

cancel Endpoint: The POST /api/upload/cancel endpoint should immediately delete the temporary files for the specified uploadId.

4.6. Security Considerations
Authentication & Authorization: All API endpoints should be protected. Only authenticated and authorized users should be able to upload files.

File Type Validation: Validate file types on the backend (e.g., using Apache Tika) to prevent malicious uploads, not just relying on file extensions.

File Size Limits: While supporting large files, still enforce a maximum allowed file size to prevent abuse.

Path Traversal Prevention: Sanitize file names and paths to prevent directory traversal attacks when saving files.

Antivirus Scanning: Integrate with an on-premise antivirus solution to scan uploaded files before making them accessible.

Disk Quotas: Implement disk quotas for users or applications if storage is a concern.

5. Overall System Considerations
5.1. Scalability
Horizontal Scaling (Backend): The backend should be stateless regarding the upload process. Each chunk upload is independent. The temporary storage (if on a shared NFS) and the final storage should be accessible by all Spring Boot instances. If local disk is used for temporary storage, ensure sticky sessions or a distributed temporary storage mechanism.

Load Balancer: Use a load balancer (e.g., Nginx, HAProxy) to distribute traffic across multiple Spring Boot instances.

5.2. Network Configuration
Timeout Settings: Configure timeouts on load balancers, web servers (if any, e.g., Nginx in front of Spring Boot), and Spring Boot itself to accommodate chunk uploads, but not so long that they hide actual network issues.

Bandwidth: Ensure sufficient network bandwidth between the client and the server, and between the server and the permanent storage.

5.3. Monitoring and Logging
Application Logs: Log all significant events (upload initiation, chunk received, chunk failed, file reassembled, cleanup).

Metrics: Monitor CPU, memory, disk I/O, and network usage on the server. Track upload rates, number of concurrent uploads, and success/failure rates.

Alerting: Set up alerts for failed uploads, storage space issues, or high resource utilization.

5.4. Database (Optional but Recommended)
A database (e.g., PostgreSQL, MySQL) can be used to persist metadata about ongoing uploads, such as:

uploadId

fileName

fileSize

status (e.g., IN_PROGRESS, COMPLETED, FAILED, CANCELLED)

receivedChunks (a list or bitmask of chunk numbers received)

createdAt, lastUpdated
This allows the backend to be truly stateless and enables more robust resumability and cleanup processes, especially across server restarts.

6. Conclusion
By implementing file chunking, resumable uploads, and careful consideration of on-premise infrastructure, this design provides a robust and reliable solution for handling extremely large file uploads in an Angular and Spring Boot application. The focus on temporary storage, clear API contracts, and comprehensive error handling ensures a resilient system capable of managing 12GB files efficiently.
