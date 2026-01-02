<script>
    let files = $state();
    let fileName = $state('Choose .ics file');
    let message = $state();
    let fileContent = $state('');
    let fileUploaded = $state(false);

    function handleFileChange(e) {
        if (files && files.length > 0) {
            fileName = files[0].name;
            fileUploaded = true;
            message = `File loaded: ${files[0].name}`;
            readFileContent(files[0]);
        }
    }

    function readFileContent(file) {
        const reader = new FileReader();
        reader.onload = (e) => {
            fileContent = e.target.result;
        };
        reader.onerror = () => {
            message = 'Error reading file';
        };
        reader.readAsText(file);
    }

    function convertICStoJSON(file) {
        if (!file) {
            message = 'No file selected';
            return;
        }
        // Add your conversion logic here
        message = 'Converting...';
        // Example: parse ICS content
        // const lines = fileContent.split('\n');
        // Process and convert...
    }

    function resetUpload() {
        files = null;
        fileName = 'Choose .ics file';
        fileContent = '';
        fileUploaded = false;
        message = '';
    }

    function downloadContent(format = 'json') {
        if (!fileContent) {
            message = 'No file content to download';
            return;
        }
        
        let content = fileContent;
        let type = 'text/plain';
        let extension = 'txt';
        
        if (format === 'json') {
            // Convert ICS to JSON if needed
            extension = 'json';
            type = 'application/json';
        }
        
        const blob = new Blob([content], { type });
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = `converted_file.${extension}`;
        a.click();
        URL.revokeObjectURL(url);
    }

</script>

<div class="page-container">
    <p>
        Upload your roster and import it to LogTen Pro.
    </p>
    
    {#if fileUploaded}
        <div class="file-actions">
            <button 
                class="button-secondary"
                onclick={() => {downloadContent('json')}}
            >
                Import to LogTen Pro
            </button>
        </div>
        {#if fileContent}
            <div class="preview-container">
                <h3>File Preview</h3>
                <textarea readonly rows="10" value={fileContent}></textarea>
            </div>
        {/if}
    {:else}
        <div class="upload-container">
            <input 
                type="file" 
                bind:files
                onchange={handleFileChange}
                accept=".ics,.csv,.txt,text/calendar,text/csv" 
                id="file" 
                />
            <label for="file">{fileName}</label>
        </div>
    {/if}
    {#if message}
        <p class="message">{message}</p>
    {/if}
</div>

<style>
    .page-container {
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: space-around;
        gap: 2rem;
        max-width: 600px;
        width: 90%;
        padding: 2rem;
        text-align: center;
    }

    .upload-container {
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 0.5rem;
    }

    .file-actions {
        display: flex;
        flex-direction: column;
        gap: 0.5rem;
        align-items: center;
        width: 100%;
    }

    .preview-container {
        width: 100%;
        margin-top: 1rem;
    }

    .preview-container h3 {
        margin: 0 0 0.5rem 0;
        color: var(--color-text-primary);
    }

    textarea {
        width: 100%;
        padding: 0.5rem;
        border: 1px solid #ccc;
        border-radius: 5px;
        font-family: monospace;
        font-size: 0.9rem;
        resize: vertical;
    }

    p {
        font-size: 1.1rem;
        color: var(--color-text-secondary);
    }

    p.message {
        margin: 0;
        color: #f15d22;
        font-weight: 500;
    }

    [type="file"] {
        height: 0;
        overflow: hidden;
        width: 0;
    }

    [type="file"] + label {
        background: #f15d22;
        border: none;
        border-radius: 5px;
        color: #fff;
        cursor: pointer;
        display: inline-block;
        font-family: 'Rubik', sans-serif;
        font-size: inherit;
        font-weight: 500;
        margin-bottom: 1rem;
        outline: none;
        padding: 1rem 50px;
        position: relative;
        transition: all 0.3s;
        vertical-align: middle;
    }

    [type="file"] + label:hover {
        background: #d94a1a;
        transform: scale(1.05);
    }

    .button-primary,
    .button-secondary {
        background: #f15d22;
        border: none;
        border-radius: 5px;
        color: #fff;
        cursor: pointer;
        font-family: 'Rubik', sans-serif;
        font-size: inherit;
        font-weight: 500;
        outline: none;
        padding: 0.75rem 30px;
        transition: all 0.3s;
        vertical-align: middle;
    }

    .button-primary:hover,
    .button-secondary:hover {
        background: #d94a1a;
        transform: scale(1.05);
    }

    .button-secondary {
        background: #6c757d;
        margin-top: 0.5rem;
    }

    .button-secondary:hover {
        background: #5a6268;
    }

</style>