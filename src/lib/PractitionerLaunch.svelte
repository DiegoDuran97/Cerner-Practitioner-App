<script lang="ts">
    import { onMount } from "svelte";
    import axios from 'axios';
    import PatientBanner from './PatientBanner.svelte';
    import ObservationViewer from "./ObservationViewer.svelte";

    let clientId = 'de47446c-03b5-4c4c-9ac9-04a236270baa';
    let redirectUri = 'http://localhost:5173';

    let baseUrl: string | undefined;

    interface TokenResponse {
        "access_token": string;
        "patient": string;
        "scope": string;
        "need_patient_banner": boolean;
        "id_token": string;
        "smart_style_url": string;
        "expires_in": number;
    }

    let token: TokenResponse | undefined;

    const LOCALSTORAGE_TOKEN_ENDPOINT = 'tokenEndpoint';
    const LOCALSTORAGE_ISS = 'iss';
    const LOCALSTORAGE_LAUNCH = 'launch';
    const LOCALSTORAGE_TOKEN_JSON = 'token';

    const constructAuthUrl = (authorizationEndpoint: string, launch: string, iss: string) => {
        const url = new URL(authorizationEndpoint);
        url.searchParams.set('client_id', clientId);
        url.searchParams.set('redirect_uri', redirectUri);
        url.searchParams.set('scope', 'openid fhirUser launch user/Observation.read user/Observation.write user/Patient.read');
        url.searchParams.set('response_type', 'code');
        url.searchParams.set('aud', iss);
        url.searchParams.set('launch', launch);
        return url.href;
    };

    const makeTokenRequest = async (code: string) => {
        const tokenEndpoint = localStorage.getItem(LOCALSTORAGE_TOKEN_ENDPOINT);
        if (!tokenEndpoint) {
            throw new Error('Token Endpoint could not be found in localStorage');
        }

        console.log('Token Endpoint:', tokenEndpoint);

        const form = new URLSearchParams();
        form.set('grant_type', 'authorization_code');
        form.set('code', code);
        form.set('redirect_uri', redirectUri);
        form.set('client_id', clientId);

        try {
            const tokenResponse = await axios.post(tokenEndpoint, form, {
                headers: {
                    'Content-Type': 'application/x-www-form-urlencoded'
                }
            });
            localStorage.removeItem(LOCALSTORAGE_TOKEN_ENDPOINT);
            return tokenResponse.data;
        } catch (error: unknown) {
            if (error instanceof Error) {
                console.error('Error making token request:', error.message);
            } else {
                console.error('Unknown error occurred while making token request:', error);
            }
            throw error;
        }
    };

    onMount(async () => {
        const launchUrl = new URL(window.location.href);
        let code = launchUrl.searchParams.get('code');
        let iss = launchUrl.searchParams.get('iss');
        let launch = launchUrl.searchParams.get('launch');

        console.log('Current URL:', window.location.href);
        console.log('Code:', code);
        console.log('ISS:', iss);
        console.log('Launch:', launch);

        if (iss) localStorage.setItem(LOCALSTORAGE_ISS, iss);
        if (launch) localStorage.setItem(LOCALSTORAGE_LAUNCH, launch);

        const storedIss = localStorage.getItem(LOCALSTORAGE_ISS) || undefined;  // Handle null case
        const storedLaunch = localStorage.getItem(LOCALSTORAGE_LAUNCH) || undefined;  // Handle null case

        baseUrl = storedIss;

        const tokenJSON = localStorage.getItem(LOCALSTORAGE_TOKEN_JSON);

        if (code) {
            try {
                const tokenFromCerner = await makeTokenRequest(code);
                console.log('Token from Cerner:', tokenFromCerner);
                token = tokenFromCerner;
                localStorage.setItem(LOCALSTORAGE_TOKEN_JSON, JSON.stringify(token));
                return;
            } catch (error: unknown) {
                if (error instanceof Error) {
                    console.error('Error fetching token:', error.message);
                } else {
                    console.error('Unknown error occurred while fetching token:', error);
                }
            }
        }

        if (!storedIss || !storedLaunch) {
            console.error('ISS or Launch parameters not found');
            return;
        }

        try {
            const SmartConfigurationResponse = await axios.get(`${storedIss}/.well-known/smart-configuration`);
            const SmartConfiguration = SmartConfigurationResponse.data;

            console.log('SMART Configuration Response:', SmartConfiguration);

            const authorizationEndpoint = SmartConfiguration.authorization_endpoint as string;
            const tokenEndpoint = SmartConfiguration.token_endpoint as string;

            console.log('Authorization Endpoint:', authorizationEndpoint);
            console.log('Token Endpoint:', tokenEndpoint);

            if (!authorizationEndpoint || !tokenEndpoint) {
                throw new Error('Authorization or Token Endpoint is missing');
            }

            localStorage.setItem(LOCALSTORAGE_TOKEN_ENDPOINT, tokenEndpoint);

            if (storedLaunch && storedIss) {
                const redirectUrl = constructAuthUrl(authorizationEndpoint, storedLaunch, storedIss);
                window.location.href = redirectUrl;
            }
        } catch (error: unknown) {
            if (error instanceof Error) {
                console.error('Error fetching Smart Configuration:', error.message);
            } else {
                console.error('Unknown error occurred while fetching Smart Configuration:', error);
            }
        }
    });
</script>

{#if !token}
    Loading...
{:else}
    {#if token?.need_patient_banner && baseUrl}
        <PatientBanner {baseUrl} accessToken={token.access_token} patientId={token.patient}></PatientBanner>
        <ObservationViewer {baseUrl} accessToken={token.access_token} patientId={token.patient}></ObservationViewer>
    {/if}
{/if}
