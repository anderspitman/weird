<script lang="ts">
	import getPkce from 'oauth-pkce';
	import { env } from '$env/dynamic/public';
	import { getUserInfo } from '$lib/rauthy';
	import { ProgressRadial } from '@skeletonlabs/skeleton';
	import type { PageData } from './$types';

	const { data }: { data: PageData } = $props();
	const profile = data.profile;
	const userName = profile?.username?.split('@')[0];
	const displayName = profile?.display_name;
	const userInfo = getUserInfo();

	const publicUrl = new URL(env.PUBLIC_URL);

	let customDomain = $state(profile?.custom_domain || '');
	let domainValid = $state(false);
	let domainStatus = $state('Domain not set');
	let domainStatusColor = $state('hsla(0, 100%, 0%, 0.5)');
	let domainReCheck = $state(0);

	setInterval(() => {
		domainReCheck = domainReCheck + 1;
	}, 5000);

	$effect(() => {
		(async () => {
			domainReCheck;

			if (customDomain != '') {
				if (customDomain.endsWith(env.PUBLIC_DOMAIN)) {
					domainStatus = 'Cannot use instance domain';
					domainValid = false;
					domainStatusColor = 'red';
					return;
				}

				let resp;
				try {
					resp = await fetch(
						`/account/${userInfo?.id}/custom-domain/check/${customDomain}/${data.dnsChallenge}`
					);
				} catch (_) {}
				if (resp?.status == 200) {
					domainStatus = 'Domain validated';
					domainValid = true;
					domainStatusColor = 'green';
				} else {
					domainStatus =
						'Domain check unsuccessful. If you already set the DNS correctly you may just need to wait for it to propagate.';
					domainValid = false;
					domainStatusColor = 'red';
				}
			} else {
				domainStatus = '';
				domainValid = true;
			}
		})();
	});

	const siteUrl = profile?.custom_domain
		? `https://${profile.custom_domain}`
		: `${publicUrl.protocol}//${userName}.${env.PUBLIC_DOMAIN}`;

        const AddDomainTakingNames = async (e: Event) => {
                e.preventDefault();

                const [challenge, verifier] = await new Promise((resolve, reject) => {
                        getPkce(64, (error, { challenge, verifier }) => {
                                if (!error) {
                                        resolve([challenge, verifier]);
                                }
                                else {
                                        reject(error);
                                }
                        });
                });

                const state = crypto.randomUUID();
                localStorage.setItem('namedrop_code_verifier_' + state, verifier);
                const url = new URL(`${env.PUBLIC_NAMEDROP_URI}/authorize`);
                url.searchParams.set('client_id', window.location.origin);
                url.searchParams.set('redirect_uri', window.location.origin + '/account/namedrop/callback');
                url.searchParams.set('response_type', 'code');
                url.searchParams.set('scope', 'subdomain');
                url.searchParams.set('code_challenge', challenge);
                url.searchParams.set('code_challenge_method', 'S256');
                url.searchParams.set('state', state);
                window.location.href = url.href;
        };
</script>

<svelte:head>
	<title>Profile | {env.PUBLIC_INSTANCE_NAME}</title>
</svelte:head>

{#if userInfo && profile}
	<main class="flex items-start justify-center gap-5 pt-12">
		<!-- TODO: Abstract This Sidebar into a Nested Layout Shared with the Profile Page -->
		<section class="card px-5 py-4">
			<h2 class="text-lg font-bold">{displayName}</h2>

			<ol class="list-nav mt-4 flex flex-col gap-2">
				<li>
					<a href="/auth/v1/account"> Profile </a>
				</li>
				<li>
					<a href={`/account/${userInfo.id}/custom-domain`} class="variant-outline">
						Site Generation
					</a>
				</li>
			</ol>
		</section>

		<div class="card flex w-[600px] max-w-[90%] flex-col gap-4 p-8 text-xl">
			<h1 class="my-3 text-3xl font-bold">Site Generation</h1>

			{#if profile.username}
				<div class="ml-4 flex items-center gap-2">
					<strong>Your Site:</strong> <a href={siteUrl} class="text-base underline">{siteUrl}</a>
				</div>

				<h2 class="mt-4 text-xl font-bold">Custom Domain</h2>

				<div class="prose flex flex-col gap-2 text-base">
					<p>
						By default your site is hosted as a sub-domain of Weird.one, but you can also use your
						own custom domain.
					</p>
					<p>
						Before setting your custom domain below, you must configure your DNS provider: add a <code
							>CNAME</code
						>
						record for your desired domain, and set it to
						<code>{env.PUBLIC_DOMAIN}</code>. It may take some time before the update is active.
					</p>
					<p>
						After you configure your DNS, you can enter your custom domain below. This page will
						check that the update has been applied properly and will let you save once it is
						complete.
					</p>
					{#if data.serverIp}
						<p>
							If you are unable to use a <code>CNAME</code> record, you may use an <code>A</code>
							record and point it to <code>{data.serverIp}</code>
						</p>
					{/if}

					<form method="post">
						<label>
							<span class="text-lg font-bold">Custom Domain</span>
							<input name="custom_domain" class="input" bind:value={customDomain} />
						</label>

						<div class="ml-4 mt-4 flex items-center gap-3">
							{#if customDomain != '' && !domainValid}
								<ProgressRadial width="w-8" />
							{/if}
							<div class="flex-grow" style="color: {domainStatusColor}">
								{domainStatus}
							</div>
							<button disabled={!domainValid} class="variant-ghost btn"> Save </button>
						</div>
					</form>
                                        <button type="button" class="variant-ghost-surface btn" onclick={AddDomainTakingNames} >
                                                Connect Domain with TakingNames
                                        </button>
				</div>
			{:else}
				<p class="text-lg">You must set a username in the profile page to generate a website.</p>
			{/if}
		</div>
	</main>
{/if}

<style>
	code {
		background: hsla(0, 0%, 0%, 0.1);
		padding: 0.1em;
		border-radius: 2px;
	}
</style>
