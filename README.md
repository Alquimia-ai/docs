# Alquimia Runtime Documentation

This repository contains the documentation for Alquimia Runtime, a powerful platform for dynamic AI assistants.

## Features

- **Dynamic AI Assistants**: Interact with intelligent AI assistants through our chat API
- **Session Management**: Maintain conversation context with session-based interactions
- **File Attachments**: Support for file uploads and document processing
- **Real-time Responses**: Get instant responses from AI assistants
- **Assistant Configuration**: Retrieve detailed configuration and personality settings for each assistant

## Development

Install the [Mintlify CLI](https://www.npmjs.com/package/mintlify) to preview the documentation changes locally. To install, use the following command

```
npm i -g mintlify
```

Run the following command at the root of your documentation (where docs.json is)

```
mintlify dev
```

## API Endpoints

- **POST /infer/chat/{assistant_id}**: Start a conversation with a specific AI assistant
- **GET /chat/{assistant_id}**: Get detailed configuration information for a specific assistant

## Publishing Changes

Install our Github App to auto propagate changes from your repo to your deployment. Changes will be deployed to production automatically after pushing to the default branch. Find the link to install on your dashboard. 

#### Troubleshooting

- Mintlify dev isn't running - Run `mintlify install` it'll re-install dependencies.
- Page loads as a 404 - Make sure you are running in a folder with `docs.json`
