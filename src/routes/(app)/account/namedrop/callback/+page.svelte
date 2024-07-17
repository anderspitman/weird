<script lang="ts">
	import { onMount } from 'svelte';
	import { env } from '$env/dynamic/public';
	import { getUserInfo } from '$lib/rauthy';

        let domainStatus = '';
	const userInfo = getUserInfo();

        onMount(async () => {

                console.log(userInfo);

                const url = new URL(window.location.href);

                const state = url.searchParams.get('state');
                if (!state) {
                        console.error("Missing state param");
                        return;
                }

                const code = url.searchParams.get('code');
                if (!code) {
                        console.error("Missing code param");
                        return;
                }

                const codeVerifier = localStorage.getItem("namedrop_code_verifier_" + state);
                if (!codeVerifier) {
                        console.error("No valid code verifier");
                        return;
                }

                const response = await fetch(`${env.PUBLIC_NAMEDROP_URI}/token`, {
                        method: 'POST',
                        headers: {
                                'Content-Type': 'application/x-www-form-urlencoded',
                        },
                        body: new URLSearchParams({
                                client_id: window.location.origin,
                                grant_type: 'authorization_code',
                                code,
                                redirect_uri: window.location.origin + '/account/namedrop/callback',
                                code_verifier: codeVerifier,
                        }),
                });

                const tokenData = await response.json();

                const host = tokenData.permissions[0].host;
                const domain = tokenData.permissions[0].domain;
                const token = tokenData.access_token;

                const recordType = host ? 'CNAME' : 'ANAME';
                const recordValue = env.PUBLIC_DOMAIN.startsWith('localhost') ? 'weird.one' : env.PUBLIC_DOMAIN;

                const recRes = await fetch(`${env.PUBLIC_NAMEDROP_URI}/records?access_token=${token}`, {
                        method: 'POST',
                        body: JSON.stringify({
                                domain,
                                host,
                                type: recordType,
                                value: recordValue,
                                ttl: 300,
                        }),
                });

                if (recRes.ok) {
                        const fqdn = `${host}.${domain}`;
                        window.location.href = `/account/${userInfo.id}/custom-domain?domain=${fqdn}`;
                }
                else {
                        domainStatus = "Failed to set domain";
                }
        });

</script>

<svelte:head>
	<title>
	</title>
</svelte:head>

<main>
        <div class="card flex w-[600px] max-w-[90%] flex-col gap-4 p-8 text-xl">
                <p>
                        {domainStatus}
                </p>
        </div>
</main>
