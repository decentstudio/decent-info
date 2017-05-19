# decent-info
Knowledge base and documentation. Overarching concerns.

## Slash Commands
- Must respond within 3 seconds at least with a 200 status (if want to send message later from event handler)
- Set response_type to 'ephemeral' to make the slash command the user typed and the response hidden to other users, and 'in_channel' to display the command and response to all users in the channel
- If you're not ready to respond to an incoming command but still want to acknowledge the user's action by having their slash command displayed within the channel, respond to your URL's invocation with a simplified JSON response containing only the response_type field set to in_channel: {"response_type": "in_channel"}.
- If your command doesn't need to post anything back (either privately or publicly), respond with an empty HTTP 200 response. Best to use this only if the nature of your command makes it obvious that no response is necessary or desired. Even a simple "Got it!" ephemeral response is better than nothing.
- Provide command usage when /command help is used
- Turn on entity resolution for usernames, channels, and links by flipping the toggle in your slash command's configuration dialog. Always work with user IDs and channel IDs.
- Before submitting a command to your server, Slack will occasionally send your command URLs a simple POST request to verify the certificate. These requests will include a parameter ssl_check set to 1. Mostly, you may ignore these requests, but please do respond with a HTTP 200 OK.
