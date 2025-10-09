# OpenAPI Specification

This directory contains the OpenAPI 3.0.3 specification for the eDiscovery API.

## Files

- `openapi.yaml` - Complete OpenAPI specification for all API endpoints

## API Overview

The eDiscovery API provides endpoints for:

1. **Authentication** - User signup and login with JWT token-based authentication
2. **Users** - WhatsApp device/user management
3. **Chats** - Chat list retrieval for legal hold devices
4. **Messages** - Paginated message retrieval from chats
5. **QR Code** - WhatsApp QR code generation for device pairing

## Endpoints

### Authentication
- `POST /api/signup` - Create a new user account
- `POST /api/login` - Authenticate and receive JWT token

### Users & Devices
- `GET /api/users` - Get all paired WhatsApp devices

### Chats
- `GET /api/{lhid}/chats` - Get all chats for a specific device

### Messages
- `GET /api/{lhid}/chats/{chatid}/messages` - Get messages from a chat (paginated)

### QR Code
- `GET /api/code` - Generate WhatsApp QR code for device pairing

## Authentication

Most endpoints require JWT authentication via HTTP-only cookies. The authentication flow is:

1. Sign up using `POST /api/signup`
2. Log in using `POST /api/login` to receive a JWT token in a cookie
3. Use the token automatically in subsequent requests (valid for 24 hours)

## Viewing the Specification

You can view and interact with this API specification using:

- [Swagger Editor](https://editor.swagger.io/) - Import the `openapi.yaml` file
- [Swagger UI](https://swagger.io/tools/swagger-ui/) - Host the spec for interactive documentation
- [Redoc](https://redocly.com/redoc/) - Generate beautiful API documentation

## Validation

The OpenAPI specification has been validated for correct YAML syntax and OAS 3.0.3 compliance.
