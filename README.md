# chat-exemple
Example of using search APIs with Svelte components 

## Component Summary 

This component is a floating chat widget designed to assist users in finding real estate properties. It manages a conversational interface that interacts with an API to:
	•	Collect user input and display chat history.
	•	Initiate or resume chat sessions using a chatId (stored in URL parameters).
	•	Detect and validate email addresses to send property listings.
	•	Handle cases where no properties are found and allow users to restart the search.
	•	Automatically scroll to the bottom when new messages are added.
	•	Present a toggleable UI with a minimized button and a full chat box interface.

The component supports features like:
	•	Email validation and follow-up messaging.
	•	Managing session state (new session, ongoing conversation, results found).
	•	A callback function to return found PropertyFullType[] objects to the parent.
	•	Adaptive input behavior depending on whether it expects a message or email.
