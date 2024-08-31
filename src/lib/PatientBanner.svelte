<script lang="ts">
    import axios from "axios";
    import type { Patient } from "fhir/r4";
    import { onMount } from "svelte";

    export let accessToken: string;
    export let baseUrl: string | undefined;
    export let patientId: string;

    let patientResource: Patient | undefined = undefined;
    let error: string | undefined = undefined;

    onMount(async () => {
        if (!baseUrl) {
            error = 'Base URL is missing';
            console.error('Base URL is not provided.');
            return;
        }

        try {
            console.log("Fetching patient data from:", `${baseUrl}/Patient/${patientId}`);
            const response = await axios.get<Patient>(`${baseUrl}/Patient/${patientId}`, {
                headers: {
                    'Authorization': `Bearer ${accessToken}`
                }
            });

            if (typeof response.data === 'object' && response.data) {
                patientResource = response.data;
                console.log("Received patient data:", patientResource);
            } else {
                throw new Error('Unexpected response format');
            }
        } catch (err: unknown) {
            error = 'Failed to fetch patient data';
            console.error('Error fetching patient data:', err);

            if (err instanceof Error) {
                const axiosError = err as any;
                if (axiosError.response) {
                    console.error('Server responded with status:', axiosError.response.status);
                    console.error('Response data:', axiosError.response.data);
                }
            }
        }
    });
</script>

<div class="p-3 flex gap-5 bg-slate-700 text-white">
    {#if error}
        <p>{error}</p>
    {:else if patientResource}
        <p>
            Name: {patientResource.name?.[0]?.text || `${patientResource.name?.[0]?.given?.join(' ')} ${patientResource.name?.[0]?.family}` || 'Not available'}
        </p>
        <p>
            Date of Birth: {patientResource.birthDate || 'Not available'}
        </p>
        <p>
            Gender: {patientResource.gender || 'Not available'}
        </p>
    {:else}
        Loading Patient ...    
    {/if}
</div>
