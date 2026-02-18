<script>
    import { icsToJson } from "ics-to-json";

    let files = $state();
    let fileName = $state('Choose .ics file');
    let message = $state();
    let fileContent = $state('');
    let fileUploaded = $state(false);
    let loading = $state(false);

    let payload = $state({
        "entities": [],
        "metadata": {
            "dateAndTimeFormat": "MM/dd/yyyy HH:mm",
            "timesAreZulu": true,
            "application": "ICS to LogTen Pro Converter",
            "dateFormat": "MM/dd/yyyy",
            "shouldApplyAutoFillTimes": true,
            "version": "1.0"
    }});

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

    function calSummaryConverter(summary) {

        function extractFlightDetails(summary) {
            // Extract flight number
            const flightNumberMatch = summary.match(/CX\s?\d+/);

            // Extract departure IATA (three letters before the second '-')
            const firstDashIndex = summary.indexOf('-');
            const secondDashIndex = summary.indexOf('-', firstDashIndex + 1);
            const departureIATA = secondDashIndex > 2 ? summary.substring(secondDashIndex - 3, secondDashIndex) : null;
            const arrivalIATA = secondDashIndex >= 0 && secondDashIndex + 3 < summary.length ? summary.substring(secondDashIndex + 1, secondDashIndex + 4) : null;

            return {
                flightNumber: flightNumberMatch ? flightNumberMatch[0].replace(/\s/g, '') : null,
                departureIATA,
                arrivalIATA
            };
        }

        if (summary.includes('/')) {
            const flightParts = summary.split('/');
            const formattedParts = [];

            for (const part of flightParts) {
                formattedParts.push(extractFlightDetails(part));
            }
            
            return formattedParts;

        }
        else {
            return extractFlightDetails(summary);
        };
        
    }

    function dateConverter(icsDate) {
        // Convert ICS date format to JavaScript Date object
        const year = icsDate.substring(0, 4);
        const month = icsDate.substring(4, 6) - 1; // Months are zero-based in JS
        const day = icsDate.substring(6, 8);
        const hours = icsDate.substring(9, 11);
        const minutes = icsDate.substring(11, 13);
        const seconds = icsDate.substring(13, 15);

        return `${month + 1}/${day}/${year}`; // MM/DD/YYYY
    }

    function dateTimeConverter(icsDate, startTimeChange = 0) {

        const date = new Date(
            icsDate.substring(0, 4), // Year
            icsDate.substring(4, 6) - 1, // Month (zero-based)
            icsDate.substring(6, 8), // Day
            icsDate.substring(9, 11), // Hours
            icsDate.substring(11, 13), // Minutes
            icsDate.substring(13, 15) // Seconds
        );


        // Adjust date by minutes if needed
        const adjustedDate = new Date(date.getTime() + startTimeChange * 60000); // Convert minutes to milliseconds
    
        // Convert adjusted date back to desired format
        const year = adjustedDate.getFullYear();
        const month = adjustedDate.getMonth(); // Months are zero-based in JS
        const day = adjustedDate.getDate();
        const hours = adjustedDate.getHours();
        const minutes = adjustedDate.getMinutes();
        const seconds = adjustedDate.getSeconds();

        return `${month + 1}/${day}/${year} ${hours}:${minutes}`; // MM/DD/YYYY HH:mm
    }

    function convertICStoJSON() {
        loading = true;

        // Add your conversion logic here
        message = 'Converting...';

        const jsonICSData = icsToJson(fileContent);

        // Add converted data to payload
        for (const event of jsonICSData) {
            
            if (/CX|[ANTBS]\s?\d{4}/.test(event.summary)) {
                const summary = calSummaryConverter(event.summary);
                for (const [index, flightSector] of (Array.isArray(summary) ? summary : [summary]).entries()) {
                    console.log(dateTimeConverter(event.startDate))

                    const startTimeChange = event.summary.includes('CX') ? (flightSector.departureIATA == 'HKG' ? -70 : -60) : 0; 
                    const isIntegratedPattern = summary.length > 0 ? true : false; // Check if the summary contains multiple flight sectors

                    console.log('sector number:', index, isIntegratedPattern);

                    // TODO - Handle integrated patterns with multiple sectors (e.g., CX flights with same flight number but different departure/arrival times)

                    const arrivalDateTime = isIntegratedPattern && index < summary.length - 1 ? null : dateTimeConverter(event.endDate);
                    // TODO - crashing coz you cant pass null for arrival time for integrated patterns, need to find a workaround for this (maybe set it to departure time + 1hr or something and then user can adjust it in LogTen Pro if needed?)
                    payload.entities.push({
                        "flight_flightDate": dateConverter(event.startDate),
                        "flight_flightDutyStartTime": index > 0 ? null : dateTimeConverter(event.startDate),
                        "flight_flightDutyEndTime": arrivalDateTime,
                        "flight_scheduledDepartureTime": index > 0 ? null : dateTimeConverter(event.startDate, startTimeChange), // Adjust departure time for CX flights
                        "flight_scheduledArrivalTime": arrivalDateTime,
                        "flight_flightNumber": flightSector.flightNumber,
                        "entity_name": "Flight",
                        "flight_type": event.summary.includes('CX') ? 0 : 3,    
                        "flight_from": flightSector.departureIATA,
                        "flight_to": flightSector.arrivalIATA,
                    });
                }
            }
        }

        window.location.href = 'logten://v2/addEntities?package=' + encodeURIComponent(JSON.stringify(payload));

        console.log('Final payload:', payload);
    }

    function resetUpload() {
        files = null;
        fileName = 'Choose .ics file';
        fileContent = '';
        fileUploaded = false;
        message = '';
    }
</script>

<div class="page-container">
    <p>
        Upload your roster and import it to LogTen Pro.
    </p>
    
    {#if fileUploaded}
        {#if !loading}
            <div class="file-actions">
                <button 
                    class="button-primary"
                    onclick={() => {convertICStoJSON()}}
                >
                    Import to LogTen Pro
                </button>
            </div>
        {:else}
            <div class="file-actions">
                Thinking
            </div>
        {/if}
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
                accept=".ics,text/calendar" 
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

    .button-primary {
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

    .button-primary:hover {
        background: #d94a1a;
        transform: scale(1.05);
    }

</style>