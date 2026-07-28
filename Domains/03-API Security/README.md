# API Notes

Personal notes and reference documentation for working with this API.

## Table of Contents

- [Overview](#overview)
- [Authentication](#authentication)
- [Base URL](#base-url)
- [Endpoints](#endpoints)
- [Request/Response Examples](#requestresponse-examples)
- [Error Handling](#error-handling)
- [Rate Limits](#rate-limits)
- [Notes & Gotchas](#notes--gotchas)
- [Useful Links](#useful-links)

## Overview

Brief description of what this API does and why you're using it.

## Authentication

```
Authorization: Bearer <your_api_key>
```

- How to get an API key:
- Where keys are stored/rotated:
- Token expiry:

## Base URL

```
https://api.example.com/v1
```

## Endpoints

| Method | Endpoint       | Description         |
|--------|----------------|----------------------|
| GET    | `/resource`    | List resources       |
| GET    | `/resource/:id`| Get single resource  |
| POST   | `/resource`    | Create resource      |
| PUT    | `/resource/:id`| Update resource      |
| DELETE | `/resource/:id`| Delete resource      |

## Request/Response Examples

### Example Request

```bash
curl -X GET "https://api.example.com/v1/resource" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

### Example Response

```json
{
  "id": "123",
  "name": "example",
  "status": "active"
}
```

## Error Handling

| Status Code | Meaning              |
|-------------|----------------------|
| 400         | Bad Request           |
| 401         | Unauthorized          |
| 403         | Forbidden             |
| 404         | Not Found             |
| 429         | Too Many Requests     |
| 500         | Server Error          |

## Rate Limits

- Requests per minute:
- Retry strategy:

## Notes & Gotchas

- 
- 
- 

## Useful Links

- [Official API Docs](#)
- [Postman Collection](#)
- [Changelog](#)
