# REST API Standard

A **REST (Representational State Transfer)** API is an architectural style for building web services that use HTTP to access and manipulate resources, typically in a stateless, client-server setup.

Author

Benedict Thekkel

## 🔧 1. **What is a REST API?**

### 🌐 Key Characteristics:

| Principle | Description |
|----|----|
| **Stateless** | Each request contains all information needed (no session state on server). |
| **Client-server** | Frontend and backend are decoupled. |
| **Cacheable** | Responses can be cached where appropriate. |
| **Uniform interface** | Standardized resource access via HTTP verbs. |
| **Layered system** | Intermediary layers (e.g., load balancers, caches) are allowed. |
| **Code on demand** | (Optional) Clients can download and execute code from the server. |

------------------------------------------------------------------------

## 🚀 2. **Standard HTTP Methods in REST**

| HTTP Method | Usage            | Example             |
|-------------|------------------|---------------------|
| `GET`       | Read             | `GET /users/123`    |
| `POST`      | Create           | `POST /users`       |
| `PUT`       | Replace (full)   | `PUT /users/123`    |
| `PATCH`     | Update (partial) | `PATCH /users/123`  |
| `DELETE`    | Delete           | `DELETE /users/123` |

------------------------------------------------------------------------

## 📂 3. **URI Naming Conventions**

- Use **nouns**, not verbs: ✅ `GET /users` ❌ `GET /getUsers`

- Use **plural** resource names: ✅ `/books` ❌ `/book`

- Nest relationships: `GET /users/123/orders` → orders for a user

- Use query parameters for filtering: `/books?genre=sci-fi&author=asimov`

------------------------------------------------------------------------

## ✅ 4. **Status Codes**

| Code | Meaning               | When to Use                           |
|------|-----------------------|---------------------------------------|
| 200  | OK                    | GET, PUT, PATCH success               |
| 201  | Created               | Successful resource creation (POST)   |
| 204  | No Content            | DELETE or PUT success with no body    |
| 400  | Bad Request           | Malformed request or validation error |
| 401  | Unauthorized          | Auth needed                           |
| 403  | Forbidden             | Auth provided but access denied       |
| 404  | Not Found             | Resource doesn’t exist                |
| 409  | Conflict              | Duplicate or conflicting state        |
| 500  | Internal Server Error | Unhandled server error                |

------------------------------------------------------------------------

## 🧪 5. **Request & Response Structure**

### JSON Request:

``` json
POST /users
{
  "email": "user@example.com",
  "password": "secure123"
}
```

### JSON Response (Success):

``` json
{
  "id": 1,
  "email": "user@example.com",
  "created_at": "2025-06-17T10:00:00Z"
}
```

### JSON Response (Error):

``` json
{
  "error": "ValidationError",
  "message": "Email already exists"
}
```

------------------------------------------------------------------------

## 🔐 6. **Authentication & Security**

- Use **HTTPS** only.
- Prefer **JWT** or **OAuth 2.0** for authentication.
- Use **rate limiting** to prevent abuse.
- Validate and sanitize all input to avoid **injection attacks**.

------------------------------------------------------------------------

## 📋 7. **Versioning**

Best options:

- URL-based: `/v1/users`
- Header-based: `Accept: application/vnd.myapi.v1+json`

> ✅ Avoid breaking changes without versioning.

------------------------------------------------------------------------

## 📊 8. **Filtering, Sorting, Pagination**

| Feature    | Query Format Example                    |
|------------|-----------------------------------------|
| Filtering  | `/products?category=clothing&color=red` |
| Sorting    | `/products?sort=-price` (descending)    |
| Pagination | `/products?page=2&page_size=50`         |

------------------------------------------------------------------------

## 🔄 9. **HATEOAS (Optional Advanced)**

Hypermedia as the Engine of Application State:

``` json
{
  "id": 123,
  "name": "Example",
  "_links": {
    "self": { "href": "/items/123" },
    "next": { "href": "/items/124" }
  }
}
```

Not widely used, but adheres to pure REST principles.

------------------------------------------------------------------------

## 🧱 10. **OpenAPI / Swagger**

Document your API:

- Generate interactive docs
- Example:

``` yaml
paths:
  /users:
    get:
      summary: Get all users
      responses:
        '200':
          description: OK
```

Tools: Swagger UI, Redoc, Postman

------------------------------------------------------------------------

## 📈 11. **Forward-Thinking & Modern Additions**

| Concept           | Description                                           |
|-------------------|-------------------------------------------------------|
| **GraphQL**       | Alternative to REST for complex querying needs        |
| **gRPC**          | High-performance, binary protocol                     |
| **REST + Events** | Use Webhooks or Server-Sent Events alongside REST     |
| **JSON:API**      | Strict REST format standard (`https://jsonapi.org/`)  |
| **Async APIs**    | For microservices, combine REST with messaging queues |

------------------------------------------------------------------------

## 🛠️ 12. **Example Project Structure**

    /api
      /v1
        users/
          views.py
          serializers.py
          urls.py
          models.py
    /tests
    /docs

------------------------------------------------------------------------

## 📌 Summary Table

| Area         | Standard Practice                         |
|--------------|-------------------------------------------|
| HTTP Methods | Use GET, POST, PUT, PATCH, DELETE         |
| URIs         | Use nouns, plurals, nested resources      |
| Auth         | JWT, OAuth2, HTTPS                        |
| Responses    | Consistent JSON + meaningful status codes |
| Docs         | Use OpenAPI/Swagger                       |
| Versioning   | URL or Header-based                       |
| Pagination   | Offset or cursor-based                    |

------------------------------------------------------------------------

Back to top
