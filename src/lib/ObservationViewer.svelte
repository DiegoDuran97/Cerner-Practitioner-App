<script lang="ts">
    import axios from 'axios';
    import { formatRelative } from 'date-fns';
    import type { OperationOutcome, Bundle, BundleEntry, Observation } from 'fhir/r4';

    export let accessToken: string;
    export let patientId: string;
    export let baseUrl: string;

    let temperature: number;

    let observationPosting = false;

    const getVitalSigns = async () => {
        try {
            const vitalSignsResponse = await axios.get<Bundle<Observation | OperationOutcome>>(`${baseUrl}/Observation`, {
                params: {
                    patient: patientId,
                    category: 'vital-signs'
                },
                headers: {
                    'Authorization': `Bearer ${accessToken}`,
                },
            });
            return vitalSignsResponse.data;
        } catch (error) {
            console.error('Error fetching vital signs:', error);
            throw error;
        }
    };

    let observationResults = getVitalSigns();

    const postTemperature = async (temperature: number) => {
        observationPosting = true;
        const temperatureResource = {
            "resourceType": "Observation",
            "status": "final",
            "category": [
                {
                    "coding": [
                        {
                            "system": "http://terminology.hl7.org/CodeSystem/observation-category",
                            "code": "vital-signs",
                            "display": "Vital Signs"
                        }
                    ],
                    "text": "Vital Signs"
                }
            ],
            "code": {
                "coding": [
                    {
                        "system": "http://loinc.org",
                        "code": "8331-1"
                    }
                ],
                "text": "Temperature Oral"
            },
            "subject": {
                "reference": `Patient/${patientId}`
            },
            "effectiveDateTime": new Date().toISOString(),
            "valueQuantity": {
                "value": temperature,
                "unit": "degC",
                "system": "http://unitsofmeasure.org",
                "code": "Cel"
            }
        };

        console.log({ temperatureResource });

        try {
            const temperatureObservationResponse = await axios.post(`${baseUrl}/Observation`, temperatureResource, {
                headers: {
                    'Authorization': `Bearer ${accessToken}`,
                },
            });
            console.log({ temperatureObservationResponse });
            observationResults = getVitalSigns();
        } catch (error) {
            console.error('Error posting temperature:', error);
        } finally {
            observationPosting = false;
        }
    };

    const getObservationDisplay = (Observation: Observation | undefined) => {
        if (!Observation) {
            return '';
        }

        const codableConcept = Observation?.valueCodeableConcept;
        if (codableConcept) {
            return Observation?.valueCodeableConcept?.text;
        }
        const isBp = Observation.component !== undefined;
        if (isBp) {
            const systolicComponent = Observation.component?.find(a => a.code?.coding?.find(b => b.code == '8480-6'));
            const systolic = systolicComponent?.valueQuantity?.value;
            const diastolicComponent = Observation.component?.find(a => a.code?.coding?.find(b => b.code == '8462-4'));
            const diastolic = diastolicComponent?.valueQuantity?.value;
            return `${systolic}/${diastolic}`;
        }
        if (!Observation?.valueQuantity?.unit) {
            return Observation?.valueQuantity?.value;
        }
        return `${Observation?.valueQuantity?.value} ${Observation?.valueQuantity?.unit}`;
    };

    const getObservationEntries = (bundle: Bundle<Observation | OperationOutcome>): BundleEntry<Observation>[] => {
        if (!bundle?.entry) {
            return [];
        }
        const results = bundle.entry.filter((entry) => entry.resource?.resourceType === 'Observation') as BundleEntry<Observation>[];
        const nonPanelResults = results.filter(entry => entry?.resource?.hasMember === undefined);
        return nonPanelResults.sort((a, b) => {
            if (a?.resource?.effectiveDateTime && b?.resource?.effectiveDateTime) {
                return new Date(b?.resource?.effectiveDateTime).getTime() - new Date(a?.resource?.effectiveDateTime).getTime();
            }
            return 0;
        });
    };
</script>

<!-- Vital Signs Section -->
<div class="mt-10 mx-auto p-6 border rounded-lg shadow-md bg-white max-w-4xl">
    <h1 class="text-3xl font-bold mb-6 text-center">Vital Signs</h1>
    <div class="my-6">
        <p class="text-black font-semibold mb-4 text-center">Create New Temperature (°C)</p>
        <form on:submit|preventDefault={() => { postTemperature(temperature) }}>
            <div class="flex items-center justify-center gap-4">
                <input 
                    step=".1" 
                    min="20" 
                    max="50" 
                    bind:value={temperature} 
                    type="number" 
                    class="border border-black p-2 rounded w-32 text-center" 
                    placeholder="Temperature"
                />
                {#if observationPosting}
                    <p class="text-gray-500">Creating Vital Sign...</p>
                {:else}
                    <button type="submit" class="bg-black text-white p-2 rounded hover:bg-gray-800 transition">Submit</button>
                {/if}
            </div>
        </form>
    </div> 
    {#await observationResults}
        <p class="text-center text-gray-500">Loading vital signs...</p>
    {:then vitalSigns}
        <table class="min-w-full divide-y divide-gray-200">
            <thead class="bg-gray-100">
                <tr>
                    <th class="px-6 py-3 text-left text-xs font-medium text-black uppercase tracking-wider">Date</th>
                    <th class="px-6 py-3 text-left text-xs font-medium text-black uppercase tracking-wider">Vital Sign</th>
                    <th class="px-6 py-3 text-left text-xs font-medium text-black uppercase tracking-wider">Value</th>
                </tr>
            </thead>
            <tbody class="bg-white divide-y divide-gray-200">
                {#each getObservationEntries(vitalSigns) as Observation}
                    <tr class="hover:bg-gray-50 transition">
                        <td class="px-6 py-4 whitespace-nowrap text-sm font-medium text-black">
                            {#if Observation?.resource?.effectiveDateTime}
                                {formatRelative(new Date(Observation?.resource?.effectiveDateTime), new Date())}
                            {/if}
                        </td>
                        <td class="px-6 py-4 whitespace-nowrap text-sm text-black">
                            {Observation.resource?.code?.text}
                        </td>
                        <td class="px-6 py-4 whitespace-nowrap text-sm text-black">
                            {getObservationDisplay(Observation?.resource)}
                        </td>
                    </tr>
                {/each}
            </tbody>
        </table>
    {/await}
</div>

<style>
    .border {
        border-color: #e5e7eb; /* Light gray border color */
    }
    .shadow-md {
        box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    }
    table {
        border-collapse: collapse;
        width: 100%;
    }
    th, td {
        border: 1px solid #e5e7eb; /* Light gray border color */
        padding: 0.75rem;
    }
    h1, th, td {
        color: black;
    }
    button:disabled {
        background-color: #d1d5db;
        cursor: not-allowed;
    }
</style>
