# Upload Logic Changes Documentation

## Overview

This document details the changes made to the stealer log parser's upload functionality, transitioning from a single combined ZIP upload to individual machine ZIP uploads.

---

## Before: Original Upload Logic

### How It Worked

1. **Processing Phase**
   - Extracted and parsed all machines from the uploaded ZIP file
   - Filtered machines based on African countries
   - Stored all African machine data to MongoDB collections

2. **Upload Phase**
   - Created **ONE filtered ZIP file** containing all African machines
   - Uploaded this single combined ZIP to DigitalOcean Spaces
   - Stored a single URL in the `raw_logs` collection

### Original Functions

```python
def create_filtered_zip(source_dir, african_machines, output_zip_path):
    """
    Created one ZIP containing all African machines
    """
    # Added all African machine folders to a single ZIP
    # Returned one ZIP file path
```

```python
async def upload_to_spaces_with_progress(zip_path, ...):
    """
    Uploaded the single combined ZIP file
    """
    # Called create_filtered_zip() once
    # Uploaded one file to Spaces
    # Stored one URL in database
```

### Database Structure (Before)

```json
{
  "_id": "...",
  "filename": "logs.zip",
  "upload_date": "2025-01-22T10:30:00",
  "spaces_url": "https://spaces.../filtered_logs.zip",
  "file_size": 150000000,
  "total_machines": 45,
  "african_machines": 12
}
```

### Limitations

- **Single point of failure**: If the combined ZIP upload failed, all machines were lost
- **Large file sizes**: Combined ZIPs could be very large (100MB+)
- **No granular access**: Couldn't access individual machines without downloading the entire ZIP
- **Difficult tracking**: Hard to identify which specific machines were uploaded
- **Bandwidth intensive**: Re-uploading required uploading all machines again

---

## After: Individual Machine Upload Logic

### How It Works Now

1. **Processing Phase** (Unchanged)
   - Extracts and parses all machines from the uploaded ZIP file
   - Filters machines based on African countries
   - Stores all African machine data to MongoDB collections

2. **Upload Phase** (Modified)
   - Creates **individual ZIP files** for each African machine
   - Uploads each machine's ZIP separately to DigitalOcean Spaces
   - Stores an array of uploaded machines with individual URLs in `raw_logs`
   - Tracks success/failure status for each machine upload

### New Functions

```python
def create_individual_machine_zip(machine_folder_path, output_zip_path):
    """
    Creates a ZIP file for a single machine
    
    Args:
        machine_folder_path: Path to the machine's folder
        output_zip_path: Where to save the ZIP file
    
    Returns:
        Path to created ZIP file or None if failed
    """
    # Creates one ZIP per machine
    # Preserves internal folder structure
    # Returns individual ZIP path
```

```python
async def upload_individual_machines_to_spaces(
    source_dir, 
    african_machines, 
    websocket, 
    s3_client, 
    bucket_name, 
    spaces_endpoint
):
    """
    Uploads each African machine individually
    
    Process:
    1. Iterates through each African machine
    2. Creates individual ZIP for each machine
    3. Uploads each ZIP to Spaces
    4. Tracks success/failure for each upload
    5. Returns array of uploaded machine details
    """
    # Loops through each African machine
    # Creates individual ZIPs
    # Uploads each separately
    # Returns detailed upload results
```

### Database Structure (After)

```json
{
  "_id": "...",
  "filename": "logs.zip",
  "upload_date": "2025-01-22T10:30:00",
  "total_machines": 45,
  "african_machines": 12,
  "uploaded_machines": [
    {
      "machine_name": "Machine_ABC123",
      "spaces_url": "https://spaces.../Machine_ABC123.zip",
      "file_size": 8500000,
      "upload_date": "2025-01-22T10:31:15",
      "status": "success"
    },
    {
      "machine_name": "Machine_XYZ789",
      "spaces_url": "https://spaces.../Machine_XYZ789.zip",
      "file_size": 12300000,
      "upload_date": "2025-01-22T10:31:45",
      "status": "success"
    },
    {
      "machine_name": "Machine_DEF456",
      "spaces_url": null,
      "file_size": 0,
      "upload_date": "2025-01-22T10:32:00",
      "status": "failed",
      "error": "Upload timeout"
    }
  ],
  "upload_summary": {
    "total_uploaded": 11,
    "total_failed": 1,
    "total_size": 95000000
  }
}
```

---

## Detailed Changes

### 1. New Function: `create_individual_machine_zip()`

**Purpose**: Creates a ZIP file for a single machine folder

**Implementation**:
```python
def create_individual_machine_zip(machine_folder_path, output_zip_path):
    try:
        with zipfile.ZipFile(output_zip_path, 'w', zipfile.ZIP_DEFLATED) as zipf:
            for root, dirs, files in os.walk(machine_folder_path):
                for file in files:
                    file_path = os.path.join(root, file)
                    arcname = os.path.relpath(file_path, machine_folder_path)
                    zipf.write(file_path, arcname)
        return output_zip_path
    except Exception as e:
        print(f"Error creating ZIP for {machine_folder_path}: {e}")
        return None
```

**Key Features**:
- Preserves internal folder structure of each machine
- Uses ZIP_DEFLATED for compression
- Returns None on failure for error handling
- Maintains relative paths within the ZIP

### 2. New Function: `upload_individual_machines_to_spaces()`

**Purpose**: Orchestrates the creation and upload of individual machine ZIPs

**Implementation Highlights**:
```python
async def upload_individual_machines_to_spaces(...):
    uploaded_machines = []
    failed_uploads = []
    
    for idx, machine_name in enumerate(african_machines, 1):
        # Progress tracking
        await websocket.send_json({
            "type": "progress",
            "message": f"Uploading machine {idx}/{total_machines}: {machine_name}"
        })
        
        # Create individual ZIP
        zip_path = create_individual_machine_zip(machine_folder, output_zip)
        
        # Upload to Spaces
        s3_client.upload_file(zip_path, bucket_name, spaces_key)
        
        # Track result
        uploaded_machines.append({
            "machine_name": machine_name,
            "spaces_url": spaces_url,
            "file_size": file_size,
            "upload_date": datetime.utcnow(),
            "status": "success"
        })
    
    return uploaded_machines, failed_uploads
```

**Key Features**:
- Real-time progress updates via WebSocket
- Individual error handling per machine
- Detailed tracking of success/failure
- Cleanup of temporary ZIP files
- Returns comprehensive upload results

### 3. Modified Upload Endpoint

**Changes in `/upload` endpoint**:

```python
# Before
if african_machines:
    await upload_to_spaces_with_progress(
        zip_path, 
        african_machines, 
        websocket, 
        s3_client, 
        bucket_name, 
        spaces_endpoint
    )

# After
uploaded_machines = []
failed_uploads = []

if african_machines:
    uploaded_machines, failed_uploads = await upload_individual_machines_to_spaces(
        extract_dir,
        african_machines,
        websocket,
        s3_client,
        bucket_name,
        spaces_endpoint
    )
```

### 4. Enhanced Database Storage

**Changes in `raw_logs` collection**:

```python
# Before
raw_log_entry = {
    "filename": file.filename,
    "upload_date": datetime.utcnow(),
    "spaces_url": spaces_url,
    "file_size": file_size,
    "total_machines": total_machines,
    "african_machines": len(african_machines)
}

# After
raw_log_entry = {
    "filename": file.filename,
    "upload_date": datetime.utcnow(),
    "total_machines": total_machines,
    "african_machines": len(african_machines),
    "uploaded_machines": uploaded_machines,  # Array of individual uploads
    "upload_summary": {
        "total_uploaded": len(uploaded_machines),
        "total_failed": len(failed_uploads),
        "total_size": sum(m["file_size"] for m in uploaded_machines)
    }
}
```

---

## Benefits of New Approach

### 1. **Granular Access**
- Download individual machines without fetching entire dataset
- Access specific machines by URL
- Reduced bandwidth for selective retrieval

### 2. **Better Error Handling**
- Individual machine upload failures don't affect others
- Retry logic can target specific failed machines
- Detailed error tracking per machine

### 3. **Improved Tracking**
- Know exactly which machines were uploaded successfully
- Track individual file sizes and upload times
- Monitor upload status per machine

### 4. **Scalability**
- No single large file bottleneck
- Parallel upload potential (future enhancement)
- Better for distributed storage systems

### 5. **Flexibility**
- Can implement selective re-uploads
- Easier to manage storage quotas
- Better for incremental backups

### 6. **Performance**
- Smaller individual uploads are more reliable
- Less memory usage during ZIP creation
- Faster failure recovery

---

## Migration Notes

### For Existing Data

If you have existing `raw_logs` entries with the old structure:

```python
# Old structure detection
if "spaces_url" in raw_log and "uploaded_machines" not in raw_log:
    # This is an old-style entry with combined ZIP
    # Handle accordingly
```

### Backward Compatibility

The new code doesn't break existing functionality:
- All parsing logic remains unchanged
- Database storage for machines, credentials, identities unchanged
- African country filtering logic preserved
- WebSocket progress updates maintained

---

## Testing Recommendations

### Test Cases

1. **Single Machine Upload**
   - Upload ZIP with one African machine
   - Verify individual ZIP creation and upload
   - Check database entry structure

2. **Multiple Machines Upload**
   - Upload ZIP with 10+ African machines
   - Verify all machines uploaded individually
   - Check progress tracking accuracy

3. **Mixed Success/Failure**
   - Simulate network issues during upload
   - Verify failed uploads are tracked correctly
   - Ensure successful uploads are not affected

4. **Large Machine Folders**
   - Test with machines containing 100MB+ data
   - Verify ZIP creation and upload performance
   - Check memory usage

5. **Edge Cases**
   - Empty machine folders
   - Special characters in machine names
   - Very long machine names
   - Duplicate machine names

---

## Performance Considerations

### Memory Usage

**Before**: 
- Created one large ZIP in memory
- Peak memory = size of combined ZIP

**After**:
- Creates one small ZIP at a time
- Peak memory = size of largest individual machine ZIP
- More consistent memory usage

### Upload Time

**Before**:
- One large upload (could take 5-10 minutes)
- All-or-nothing approach

**After**:
- Multiple smaller uploads (1-30 seconds each)
- Progressive completion
- Better user feedback

### Storage Organization

**Before**:
```
spaces-bucket/
  └── filtered_logs_20250122.zip (150MB)
```

**After**:
```
spaces-bucket/
  ├── Machine_ABC123_20250122.zip (8MB)
  ├── Machine_XYZ789_20250122.zip (12MB)
  ├── Machine_DEF456_20250122.zip (15MB)
  └── ... (individual machine ZIPs)
```

---

## Future Enhancements

### Potential Improvements

1. **Parallel Uploads**
   ```python
   # Use asyncio.gather for concurrent uploads
   tasks = [upload_machine(machine) for machine in african_machines]
   results = await asyncio.gather(*tasks)
   ```

2. **Retry Logic**
   ```python
   # Automatic retry for failed uploads
   for attempt in range(3):
       if upload_success:
           break
       await asyncio.sleep(2 ** attempt)
   ```

3. **Compression Options**
   ```python
   # Allow different compression levels
   zipfile.ZIP_DEFLATED  # Current
   zipfile.ZIP_BZIP2     # Better compression
   zipfile.ZIP_LZMA      # Best compression
   ```

4. **Upload Resumption**
   - Store partial upload state
   - Resume from last successful machine
   - Avoid re-uploading successful machines

5. **Batch Processing**
   - Group small machines into batches
   - Upload very large machines individually
   - Optimize based on file sizes

---

## Conclusion

The transition from single combined ZIP uploads to individual machine uploads provides:
- ✅ Better reliability and error handling
- ✅ Improved granular access and tracking
- ✅ Enhanced scalability and performance
- ✅ More flexible storage management
- ✅ Preserved all existing parsing and filtering logic

All changes are backward compatible and maintain the integrity of the original stealer log parsing functionality.
