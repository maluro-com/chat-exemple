<script lang="ts">
	import { browser } from '$app/environment';
	import { client } from '$lib/api';
	import type { PropertyFullType } from '$types/properties/properties.type';
	import { tick } from 'svelte';

	import { queryParameters } from 'sveltekit-search-params';

	const params = queryParameters();

	let {
		callback,
		messages
	}: {
		callback: (prop: PropertyFullType[] | undefined) => void;
		messages: { role: string; content: string }[];
	} = $props();

	let userInput = $state('');
	let isLoading = $state(false);
	let isWidgetOpen = $state(messages.length > 0 ? true : false); // Contrôle l'état ouvert/fermé
	let chatContainer: HTMLDivElement | null = $state(null); // Référence à l'élément DOM pour le scroll
	let getEmail = $state(false); // Contrôle l'état de la demande d'email
	let newSession = $state(false); // Contrôle l'état de la nouvelle session
	let chatId: string | undefined = $state(undefined); // ID unique pour chaque session de chat

	params.subscribe((value) => {
		console.log('params', value);
		chatId = value.chatId;
	});

	let notFound = $state(false); // Contrôle l'état de la recherche
	let askEmail = $state(false); // Contrôle l'état de la demande d'email

	function validateEmail(email: string) {
		const regex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
		return regex.test(email);
	}

	function addMessage(role: string, content: string) {
		messages = [...messages, { role, content }];
	}

	async function sendPreviousMessage() {
		if (notFound) {
			// Si la recherche n'a pas trouvé de biens
			// Enlever les 3 dernie rs messages
			messages = messages.slice(0, -3);
			if (messages.length === 0) {
				// Si tous les messages sont supprimés, réinitialise le chat
				messages = [
					{ role: 'assistant', content: 'Bonjour ! Décrivez le bien que vous rechercher' }
				];
			} else {
				// Sinon, réinitialise juste l'état
				messages = [...messages];
			}
			notFound = false; // Réinitialise l'état
			newSession = false;
		}
	}

	async function sendMessage() {
		const text = userInput.trim();
		if (!text || isLoading) return;

		notFound = false; // Réinitialise l'état

		if (validateEmail(text)) getEmail = true; // Si l'email est valide, on le demande

		if (getEmail) {
			// Si l'email est demandé
			addMessage('user', text);
			getEmail = false; // Réinitialise l'état
			isLoading = true;
			const chat = await client.POST('/chat/{sub}/sendmail', {
				fetch,
				body: { email: text },
				params: { path: { sub: chatId ?? '' } }
			});
			addMessage('assistant', "Merci, je vous envoi les biens à l'adresse : " + text);
			addMessage('assistant', 'Vous pouvez commencer une nouvelle recherche ');
			userInput = ''; // Réinitialise l'input
			newSession = true; // Réinitialise la session
			isLoading = false;
			askEmail = false; // Réinitialise l'état
			return;
		}

		if (newSession) {
			// Si une nouvelle session est demandée
			messages = [{ role: 'assistant', content: 'Bonjour ! Décrivez le bien que vous rechercher' }];
			chatId = undefined;
			newSession = false; // Réinitialise l'état
			askEmail = false; // Réinitialise l'état
		}

		addMessage('user', text);
		userInput = ''; // Réinitialise l'input
		isLoading = true;
		const chat = await client.POST('/chat/chat', {
			fetch,
			body: {
				input: text,
				chatId: chatId // Envoie l'ID de la session
			}
		});
		console.log('chat', chat);

		if (chat.error) {
			addMessage('assistant', chat.error.message);
			isLoading = false;
			return;
		}

		if (chat.data.chatId) {
			chatId = chat.data.chatId;
			params.set({ chatId: chatId }); // Met à jour l'ID de la session dans les paramètres
		} // Stocke l'ID de la session

		if (chat.data.properties && chat.data.properties.length === 0) {
			addMessage('assistant', "Je n'ai pas trouvé de biens correspondant à votre recherche.");
			addMessage('assistant', "Essayé avec d'autres critères.");
			notFound = true;
			newSession = true;
			isLoading = false;
			askEmail = false; // Réinitialise l'état
			return;
		}
		if (chat.data.properties && chat.data.properties.length < 10) {
			// Ajoute un message de confirmation
			addMessage(
				'assistant',
				"J'ai trouvé " + chat.data.properties.length + ' biens correspondant à votre recherche.'
			);
			addMessage('assistant', 'Si vous voulez les recevoir par email, saisissez le :');
			getEmail = true; // Demande l'email
			isLoading = false;
			callback(chat.data.properties as PropertyFullType[]);
		} else {
			addMessage('assistant', chat.data?.response || "Je n'ai pas compris votre demande.");
			if (messages.length > 6 && !askEmail) {
				// Ajouter la prosition de saisir son email
				askEmail = true;
				addMessage(
					'assistant',
					'Si vous voulez recevoir par mail la liste des biens déjà trouvé, indiquez le moi.'
				);
			}
			callback(chat.data?.properties);
			isLoading = false;
		}
	}

	function handleKeydown(event: { key: string; shiftKey: any; preventDefault: () => void }) {
		if (event.key === 'Enter' && !event.shiftKey) {
			event.preventDefault(); // Empêche le saut de ligne
			sendMessage();
		}
	}

	function toggleWidget() {
		isWidgetOpen = !isWidgetOpen;
		if (isWidgetOpen) {
			messages = [{ role: 'assistant', content: 'Bonjour ! Décrivez le bien que vous rechercher' }];
			chatId = undefined;
		}
	}

	// Effet pour scroller vers le bas quand un message est ajouté
	$effect(() => {
		if (browser && chatContainer && messages.length > 0) {
			// Utilise tick pour attendre que le DOM soit mis à jour AVANT de scroller
			tick().then(() => {
				chatContainer!.scrollTop = chatContainer!.scrollHeight;
			});
		}
	});
</script>

	<!-- Bouton flottant pour ouvrir/fermer -->
	{#if !isWidgetOpen}
		<button class="open-chat-button" onclick={toggleWidget} title="Ouvrir le chat"> 💬 </button>
	{/if}

	<!-- Conteneur principal du Widget -->
	{#if isWidgetOpen}
		<div class="chat-widget" role="log" aria-live="polite">
			<div class="chat-header">
				<span>Chat Assistant</span>
				<button class="close-chat" onclick={toggleWidget} title="Fermer">×</button>
			</div>
			<div class="chat-messages" bind:this={chatContainer}>
				{#each messages as message}
					<div class="message {message.role}">
						{message.content}
					</div>
				{/each}
				{#if isLoading}
					<div class="message bot typing-indicator">
						<i>Je recherche...</i>
					</div>
				{/if}
				{#if notFound && messages.length > 5}
					<button
						class="send-button"
						onclick={sendPreviousMessage}
						disabled={isLoading}
						aria-label="ou revenir à l'étape précédente"
					>
						{#if isLoading}
							Envoi...
						{:else}
							Revenir à l'étape précédente
						{/if}
					</button>
				{/if}
			</div>

			<div class="chat-input-area">
				<input
					type="text"
					class="chat-input"
					placeholder={getEmail ? 'Votre email' : 'Posez votre question...'}
					bind:value={userInput}
					onkeydown={handleKeydown}
					disabled={isLoading}
					aria-label="Votre message"
				/>
				<button
					class="send-button"
					onclick={sendMessage}
					disabled={isLoading || userInput.trim() === ''}
					aria-label="Envoyer le message"
				>
					{#if isLoading}
						Envoi...
					{:else}
						Envoyer
					{/if}
				</button>
			</div>
		</div>
	{/if}

<style>
	.open-chat-button {
		position: fixed;
		bottom: 20px;
		right: 20px;
		background-color: #007bff;
		color: white;
		border: none;
		border-radius: 50%;
		width: 60px;
		height: 60px;
		font-size: 1.8em; /* Ajusté pour mieux centrer l'emoji */
		line-height: 60px; /* Centrage vertical */
		text-align: center; /* Centrage horizontal */
		box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
		cursor: pointer;
		z-index: 999;
		transition: transform 0.2s ease-out;
	}
	.open-chat-button:hover {
		transform: scale(1.1);
	}

	/* Styles du widget lui-même */
	.chat-widget {
		position: fixed;
		bottom: 20px;
		right: 20px;
		width: 350px;
		max-width: 90%;
		height: 500px;
		max-height: 80vh;
		background-color: #fff;
		border-radius: 10px;
		box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
		display: flex;
		flex-direction: column;
		overflow: hidden;
		z-index: 1000;
		transition:
			opacity 0.3s ease,
			transform 0.3s ease; /* Ajout transition */
		/* Animation d'apparition (optionnel) */
		opacity: 1;
		transform: translateY(0);
	}

	/* Styles repris de l'exemple précédent */
	.chat-header {
		background-color: #007bff;
		color: white;
		padding: 10px 15px;
		font-weight: bold;
		border-top-left-radius: 10px;
		border-top-right-radius: 10px;
		display: flex;
		justify-content: space-between;
		align-items: center;
		flex-shrink: 0; /* Empêche le header de rétrécir */
	}

	.close-chat {
		background: none;
		border: none;
		color: white;
		font-size: 1.5em;
		cursor: pointer;
		padding: 0 5px;
		line-height: 1;
	}

	.chat-messages {
		flex-grow: 1;
		padding: 15px;
		overflow-y: auto;
		background-color: #f9f9f9;
		display: flex;
		flex-direction: column;
		gap: 10px;
	}

	.message {
		padding: 8px 12px;
		border-radius: 15px;
		max-width: 80%;
		word-wrap: break-word;
		line-height: 1.4;
	}

	.message.user {
		background-color: #007bff;
		color: white;
		align-self: flex-end;
		border-bottom-right-radius: 5px;
	}

	.message.assistant {
		background-color: #e9ecef;
		color: #333;
		align-self: flex-start;
		border-bottom-left-radius: 5px;
	}

	.message.assistant.typing-indicator {
		background-color: transparent;
		color: #888;
		padding-top: 0;
		padding-bottom: 0;
	}

	.chat-input-area {
		display: flex;
		border-top: 1px solid #ddd;
		padding: 10px;
		background-color: #fff;
		flex-shrink: 0; /* Empêche l'input de rétrécir */
	}

	.chat-input {
		flex-grow: 1;
		border: 1px solid #ccc;
		border-radius: 20px;
		padding: 10px 15px; /* Légèrement plus grand */
		margin-right: 10px;
		outline: none;
		font-size: 1em;
		line-height: 1.4; /* Pour cohérence */
	}

	.chat-input:focus {
		border-color: #007bff;
	}

	.chat-input:disabled {
		background-color: #f0f0f0;
	}

	.send-button {
		background-color: #007bff;
		color: white;
		border: none;
		border-radius: 20px;
		padding: 10px 15px; /* Cohérent avec l'input */
		cursor: pointer;
		font-weight: bold;
		transition: background-color 0.2s;
		white-space: nowrap; /* Empêche le texte de passer à la ligne */
	}

	.send-button:hover:not(:disabled) {
		background-color: #0056b3;
	}

	.send-button:disabled {
		background-color: #a0cfff;
		cursor: not-allowed;
	}
</style>
