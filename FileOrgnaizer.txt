# Global counters for moved files and removed duplicates
$global:movedFileCount = 0
$global:duplicateFileCount = 0


# Get folder path from the user
$folderPath = Read-Host "Enter the folder path you want to organize"

# Check if the folder exists
if (!(Test-Path $folderPath)) {
    Write-Host "The specified folder path does not exist. Please check the path and try again."
    exit
}

# Define file types and corresponding subfolder names
$fileCategories = @{
    'TextFiles'  = @('.txt')
    'Images'     = @('.jpg', '.jpeg', '.png', '.gif')
    'Documents'  = @('.pdf', '.docx', '.doc', '.pptx', '.xlsx')
}

# Create subfolders if they don't already exist
foreach ($category in $fileCategories.Keys) {
    $subfolderPath = Join-Path $folderPath $category
    if (!(Test-Path $subfolderPath)) {
        New-Item -Path $subfolderPath -ItemType Directory 
        Write-Host "Created subfolder: $subfolderPath"
    }
}

# Create a log file for tracking operations
$logFilePath = Join-Path $folderPath "file_organization_log.txt"
if (!(Test-Path $logFilePath)) {
    New-Item -Path $logFilePath -ItemType File 
    Write-Host "Log file created at $logFilePath"
}

function Find-UniqueFileName {
    param(
        [Parameter(Mandatory)]
        [string]$baseName,
        [Parameter(Mandatory)]
        [string]$extension,
        [Parameter(Mandatory)]
        [string]$destinationFolder
    )

    $counter = 1
    # Build unique filename with the original base name and a counter
    $uniqueFileName = "$baseName-$counter$extension"
    $uniqueDestinationPath = Join-Path $destinationFolder $uniqueFileName

    # Loop to find a unique name
    while (Test-Path $uniqueDestinationPath) {
        $counter++
        # Maintain the original base name and append a unique counter
        $uniqueFileName = "$baseName-$counter$extension"
        $uniqueDestinationPath = Join-Path $destinationFolder $uniqueFileName
    }

    return $uniqueDestinationPath
}



# Function to organize files recursively
function Organize-Files {
    param($folder)

    # Skip organizing specific folders
    $folderName = (Get-Item $folder).Name
    if ($folderName -in @("Documents", "Images", "TextFiles")) {
        Write-Host "Skipping organization of folder: $folderName"
        return
    }

    # Organize files in the current folder
    foreach ($file in Get-ChildItem -Path $folder -File) {
        $fileExtension = $file.Extension.ToLower()
        # Skip the log file itself
        if ($file.Name -eq "file_organization_log.txt") {
            continue
        }

        foreach ($category in $fileCategories.Keys) {
            if ($fileExtension -in $fileCategories[$category]) {
                $destinationFolder = Join-Path $folderPath $category

                # Check if a file with the same name already exists
                $destinationPath = Join-Path $destinationFolder $file.Name
                if (Test-Path $destinationPath) {
                    # Find a unique name for the duplicate
                    $uniqueDestinationPath = Find-UniqueFileName $file.BaseName $file.Extension $destinationFolder
                    
                    # Move the file to the unique destination
                    Move-Item -Path $file.FullName -Destination $uniqueDestinationPath
                    
                    # Log the operation
                    $logMessage = "Renamed and moved duplicate file: $($file.Name) to $uniqueDestinationPath"
                    Write-Host $logMessage
                    $logMessage | Out-File -FilePath $logFilePath -Append

                    # Increment the duplicate counter
   		 $global:duplicateFileCount++  # Using global scope
                } else {
                    # Move the file to the category
                    Move-Item -Path $file.FullName -Destination $destinationPath
                    
                    # Log the operation
                    $logMessage = "Moved $($file.Name) to $destinationFolder"
                    Write-Host $logMessage
                    $logMessage | Out-File -FilePath $logFilePath -Append
                    
                    # Increment the moved file counter
    		$global:movedFileCount++  # Using global scope
                }
                break
            }
        }
    }

    # Recursively organize files in subfolders
    foreach ($subfolder in Get-ChildItem -Path $folder -Directory) {
        Organize-Files -folder $subfolder.FullName
    }
}

# Organize files recursively
Organize-Files -folder $folderPath

# Summary
$summaryMessage = "File organization complete. $movedFileCount files moved, $duplicateFileCount duplicates handled. Logs are available at $logFilePath."
Write-Host $summaryMessage
$summaryMessage | Out-File -FilePath $logFilePath -Append