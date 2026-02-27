<script>
    import { icsToJson } from "ics-to-json";

    let files = $state();
    let fileName = $state('Choose .ics file');
    let message = $state();
    let fileContent = $state('');
    let fileUploaded = $state(false);
    let loading = $state(false);

    /* 
        Initialize payload with metadata, entities will be added to this object as we process the ICS data.
        This dict will be passed to LogTen Pro via the URL scheme, which will parse it and add the flight entries to the user's logbook.
    */
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
            message = `${files[0].name}`;
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

    // This function takes the summary string from the ICS event and extracts flight details.
    function calSummaryConverter(summary) {

        // 
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

        // Check if the summary contains multiple flight sectors (indicated by the presence of '/')
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

    // Convert ICS date format to JavaScript Date object and then format it to MM/DD/YYYY for LogTen Pro.
    function dateConverter(icsDate) {
        
        const year = icsDate.substring(0, 4);
        const month = icsDate.substring(4, 6) - 1; // Months are zero-based in JS
        const day = icsDate.substring(6, 8);
        const hours = icsDate.substring(9, 11);
        const minutes = icsDate.substring(11, 13);
        const seconds = icsDate.substring(13, 15);

        return `${month + 1}/${day}/${year}`; // MM/DD/YYYY
    }

    // This function converts ICS date and time format to JavaScript Date object, applies any necessary time adjustments, and formats it to MM/DD/YYYY HH:mm for LogTen Pro.
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

    // This function constructs the payload for LogtenPro.
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

                    const startTimeChange = event.summary.includes('CX') ? (flightSector.departureIATA == 'HKG' ? 70 : 60) : 0; 
                    const isIntegratedPattern = summary.length > 0 ? true : false; // Check if the summary contains multiple flight sectors


                    const arrivalDateTime = isIntegratedPattern && index < summary.length - 1 ? null : dateTimeConverter(event.endDate);
                    
                    // Construct the payload for this flight sector. Start with common fields, then conditionally add fields based on flight type and pattern.
                    let tempPayload = {
                        "flight_flightDate": dateConverter(event.startDate),
                        "entity_name": "Flight",
                        "flight_type": event.summary.includes('CX') ? 0 : 3,   
                    };

                    // If the flight is a sim, exclude flight number, departure and arrival airports.
                    if (tempPayload.flight_type != 3) {
                        tempPayload = {
                            ...tempPayload,
                            "flight_flightNumber": flightSector.flightNumber,
                            "flight_from": flightSector.departureIATA,
                            "flight_to": flightSector.arrivalIATA,
                            }
                        }

                    if(isIntegratedPattern) {
                        // For the first flight of the day, set the flight duty start time. 
                        if (index === 0) {
                            tempPayload = {
                                ...tempPayload,
                                "flight_flightDutyStartTime": dateTimeConverter(event.startDate),
                                "flight_scheduledDepartureTime": dateTimeConverter(event.startDate, startTimeChange),
                            }

                        // For the last flight of the day, set the flight duty end time and scheduled arrival time. 
                        } else if (index === summary.length - 1) {
                            tempPayload = {
                                ...tempPayload,
                                "flight_flightDutyEndTime": arrivalDateTime,
                                "flight_scheduledArrivalTime": arrivalDateTime,
                            }
                        }

                        // Leave times blank for intermediate flight.
                    } else {
                        // For non-integrated patterns, set the flight times.
                        tempPayload = {
                            ...tempPayload,
                            "flight_flightDutyStartTime": dateTimeConverter(event.startDate),
                            "flight_scheduledDepartureTime": dateTimeConverter(event.startDate, startTimeChange),
                            "flight_flightDutyEndTime": arrivalDateTime,
                            "flight_scheduledArrivalTime": arrivalDateTime,
                        }
                    }

                    payload.entities.push(tempPayload);
                }
            }
        }

        window.location.href = 'logten://v2/addEntities?package=' + encodeURIComponent(JSON.stringify(payload));

        console.log('Final payload:', payload);

        resetUpload();
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
                    class="upload-button"
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
        <p class="message">File Ready for Upload :</p>
        <p class="message">{message}</p>
    {/if}
</div>

<style>


</style>