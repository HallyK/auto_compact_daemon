# auto_compact_daemon
When using graphify using compact costs large amounts of tokens based on the length of the conversation. This is a backdoor fix to the compact issue, instead of using claude to write the compact to graphiffy, this daemon uses Ollama on your local GPU to summarize the exact same text, but for free. Offloading the text in a .md file.
