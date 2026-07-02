# Widget API Endpoints Documentation

## Overview

This document describes the RESTful API endpoints for managing widgets. These new endpoints replace the tab-based approach with dedicated resource endpoints.

---

## Authentication & Authorization

All endpoints require:
- **Bearer Token** authentication
- **Scope:** `widgets:update`
- **Policy:** User must have `update` permission for the widget

---

## Endpoints

### 1. Update Widget General Settings

Updates the widget's basic information.

**Endpoint:** `PUT /api/widgets/{widget}/general`

**Request Body:**
```json
{
  "name": "My Widget",
  "allowed_origins": [
    "https://example.com",
    "https://app.example.com"
  ],
  "settings": {
    "locale": "en",
    "timezone": "UTC"
  }
}
```

**Validation Rules:**
- `name`: required, string, max 255 chars, sanitized (HTML tags removed, whitespace normalized)
- `allowed_origins`: required, array
- `allowed_origins.*`: required
- `settings`: nullable, object

**Response:** `200 OK`
```json
{
  "data": {
    "id": "uuid-here",
    "name": "My Widget",
    "allowed_origins": [...],
    "settings": {...},
    "is_draft": 0,
    ...
  }
}
```

---

### 2. Update Widget Appearance

Updates the widget's theme and appearance settings.

**Endpoint:** `PUT /api/widgets/{widget}/appearance`

**Request Body:**
```json
{
  "extra_attributes": {
    "variables": {
      "primaryColor": "#007bff",
      "secondaryColor": "#6c757d",
      "fontSize": "14px"
    }
  }
}
```

**Validation Rules:**
- `extra_attributes`: nullable, object

**Response:** `200 OK`
```json
{
  "data": {
    "id": "uuid-here",
    "extra_attributes": {
      "theme": {
        "primaryColor": "#007bff",
        ...
      }
    },
    ...
  }
}
```

---

### 3. Update Widget Features

Updates enabled features for the widget.

**Endpoint:** `PUT /api/widgets/{widget}/features`

**Request Body:**
```json
{
  "features": [
    {
      "name": "chat",
      "settings": {
        "enabled": true,
        "autoOpen": false
      },
      "image": null
    },
    {
      "name": "notifications",
      "settings": {
        "sound": true
      }
    }
  ]
}
```

**Validation Rules:**
- `features`: nullable, array
- `features.*.name`: required, must exist in `widget_features` table
- `features.*.settings`: nullable, object
- `features.*.image`: nullable, file (jpeg, png, jpg, gif, svg)

**Image Upload:**
Use `multipart/form-data` when uploading images:
```
features[0][name]: chat
features[0][settings][enabled]: true
features[0][image]: (file)
```

**Response:** `200 OK`
```json
{
  "data": {
    "id": "uuid-here",
    "features": [
      {
        "id": 1,
        "name": "chat",
        "pivot": {
          "extra_attributes": {
            "enabled": true,
            "autoOpen": false
          }
        }
      }
    ],
    ...
  }
}
```

**Notes:**
- Always syncs with default features: `offline` and `branding`
- Boolean strings "true"/"false" are converted to actual booleans

---

### 4. Update Widget Fields (Prechat Survey)

Updates the prechat survey fields.

**Endpoint:** `PUT /api/widgets/{widget}/fields`

**Request Body:**
```json
{
  "fields": [
    {
      "id": 123,
      "attached": true,
      "required": true,
      "order": 0,
      "label": "Full Name",
      "values": null
    },
    {
      "id": null,
      "attached": true,
      "required": false,
      "order": 1,
      "label": "Company Name",
      "type": "text",
      "values": null
    },
    {
      "id": null,
      "attached": true,
      "required": true,
      "order": 2,
      "label": "Department",
      "type": "multiple_choice_list",
      "values": ["Sales", "Support", "Billing"]
    }
  ]
}
```

**Validation Rules:**
- `fields`: nullable, array
- `fields.*.id`: nullable, must exist in `account_fields`
- `fields.*.attached`: required, boolean
- `fields.*.required`: required, boolean
- `fields.*.order`: required, integer, min 0
- `fields.*.label`: required, string, must match `/^(?=.*[a-zA-Z])[a-zA-Z0-9 _\-]+$/`
  - Must contain at least one letter
  - Can include: letters, numbers, spaces, underscores, dashes
- `fields.*.type`: required (for new fields), enum: `text|textarea|email|number|multiple_choice_list`

**Field Name Derivation:**
The system automatically generates a `name` from the `label`:
- Converts to lowercase
- Replaces non-alphanumeric chars with underscore
- Removes leading non-letters
- Removes trailing underscores
- Must start with a letter (PHP variable naming rules)

Examples:
| Label | Generated Name |
|-------|----------------|
| `Full Name` | `full_name` |
| `Email Address` | `email_address` |
| `Phone #1` | `phone_1` |
| `123 Field` | `field` |

**Response:** `200 OK`
```json
{
  "data": {
    "id": "uuid-here",
    "fields": [...],
    ...
  }
}
```

**Error Response:** `422 Unprocessable Entity`
```json
{
  "message": "Unable to store the widget fields.",
  "errors": {
    "Company Name": "Field with the same label already exists."
  }
}
```

**Notes:**
- Existing fields: Use `id` + `attached: true` to keep them
- New fields: Omit `id` or set to `null`, system creates them
- Labels are sanitized (HTML stripped, whitespace normalized)
- Duplicate field names (derived from label) are rejected

---

### 5. Update Widget Translations

Updates translations for multiple languages.

**Endpoint:** `PUT /api/widgets/{widget}/translations`

**Request Body:**
```json
{
  "translations": {
    "en": {
      "welcome_message": "Welcome!",
      "offline_message": "We're offline",
      "placeholder": "Type your message..."
    },
    "es": {
      "welcome_message": "¡Bienvenido!",
      "offline_message": "Estamos desconectados",
      "placeholder": "Escribe tu mensaje..."
    }
  }
}
```

**Validation Rules:**
- `translations`: nullable, object

**Response:** `200 OK`
```json
{
  "data": {
    "id": "uuid-here",
    "translations": {
      "en": {...},
      "es": {...}
    },
    ...
  }
}
```

**Notes:**
- New languages automatically get default translations from config
- `null` or `"null"` string values are converted to empty strings
- Empty values are normalized

---

## Migration from Old API

### Old API (Tab-based)
```javascript
// Before
PUT /api/widgets/{widget}
{
  "tab": "general",
  "name": "My Widget",
  "allowed_origins": [...]
}
```

### New API (RESTful)
```javascript
// After
PUT /api/widgets/{widget}/general
{
  "name": "My Widget",
  "allowed_origins": [...]
}
```

### Migration Mapping

| Old Endpoint | New Endpoint |
|--------------|--------------|
| `PUT /widgets/{id}?tab=general` | `PUT /widgets/{id}/general` |
| `PUT /widgets/{id}?tab=appearance` | `PUT /widgets/{id}/appearance` |
| `PUT /widgets/{id}?tab=features` | `PUT /widgets/{id}/features` |
| `PUT /widgets/{id}?tab=fields` | `PUT /widgets/{id}/fields` |
| `PUT /widgets/{id}?tab=languages` | `PUT /widgets/{id}/translations` |

### Migration Strategy

1. **Phase 1:** Update frontend to use new endpoints (old API still works)
2. **Phase 2:** Test new endpoints thoroughly
3. **Phase 3:** Remove old tab-based API calls from frontend
4. **Phase 4:** (Optional) Deprecate old endpoint in future version

---

## Common Response Format

All successful responses follow this format:

```json
{
  "data": {
    "id": "uuid",
    "name": "Widget Name",
    "account_id": 123,
    "allowed_origins": [...],
    "settings": {...},
    "translations": {...},
    "extra_attributes": {...},
    "features": [...],
    "fields": [...],
    "is_draft": 0,
    "created_at": "2024-01-01T00:00:00.000000Z",
    "updated_at": "2024-01-01T00:00:00.000000Z"
  }
}
```

---

## Error Responses

### Validation Error (422)
```json
{
  "message": "The given data was invalid.",
  "errors": {
    "name": ["The name field is required."],
    "allowed_origins.0": ["The allowed origins.0 field is required."]
  }
}
```

### Unauthorized (403)
```json
{
  "message": "This action is unauthorized."
}
```

### Not Found (404)
```json
{
  "message": "No query results for model [App\\Models\\Widget] {id}"
}
```

---

## Frontend Implementation Examples

### React/Axios Example

```javascript
// Update general settings
const updateGeneral = async (widgetId, data) => {
  try {
    const response = await axios.put(
      `/api/widgets/${widgetId}/general`,
      {
        name: data.name,
        allowed_origins: data.allowedOrigins,
        settings: data.settings
      },
      {
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        }
      }
    );
    return response.data;
  } catch (error) {
    console.error('Update failed:', error.response?.data);
    throw error;
  }
};

// Update features with image upload
const updateFeatures = async (widgetId, features) => {
  const formData = new FormData();
  
  features.forEach((feature, index) => {
    formData.append(`features[${index}][name]`, feature.name);
    
    if (feature.settings) {
      Object.entries(feature.settings).forEach(([key, value]) => {
        formData.append(`features[${index}][settings][${key}]`, value);
      });
    }
    
    if (feature.image) {
      formData.append(`features[${index}][image]`, feature.image);
    }
  });
  
  try {
    const response = await axios.put(
      `/api/widgets/${widgetId}/features`,
      formData,
      {
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'multipart/form-data'
        }
      }
    );
    return response.data;
  } catch (error) {
    console.error('Update failed:', error.response?.data);
    throw error;
  }
};

// Update fields
const updateFields = async (widgetId, fields) => {
  try {
    const response = await axios.put(
      `/api/widgets/${widgetId}/fields`,
      { fields },
      {
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        }
      }
    );
    return response.data;
  } catch (error) {
    if (error.response?.status === 422) {
      // Handle duplicate field errors
      console.error('Duplicate fields:', error.response.data.errors);
    }
    throw error;
  }
};
```

### Vue.js Example

```javascript
// In your Vuex store or composable
export const useWidgetApi = () => {
  const updateGeneral = async (widgetId, data) => {
    return await $fetch(`/api/widgets/${widgetId}/general`, {
      method: 'PUT',
      body: data
    });
  };

  const updateAppearance = async (widgetId, data) => {
    return await $fetch(`/api/widgets/${widgetId}/appearance`, {
      method: 'PUT',
      body: data
    });
  };

  const updateTranslations = async (widgetId, translations) => {
    return await $fetch(`/api/widgets/${widgetId}/translations`, {
      method: 'PUT',
      body: { translations }
    });
  };

  return {
    updateGeneral,
    updateAppearance,
    updateTranslations
  };
};
```

---

## Testing with cURL

```bash
# Update general settings
curl -X PUT "https://api.livecaller.io/api/widgets/{widget-id}/general" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My Updated Widget",
    "allowed_origins": ["https://example.com"],
    "settings": {"locale": "en"}
  }'

# Update fields
curl -X PUT "https://api.livecaller.io/api/widgets/{widget-id}/fields" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "fields": [
      {
        "id": null,
        "attached": true,
        "required": true,
        "order": 0,
        "label": "Email",
        "type": "email"
      }
    ]
  }'

# Update features with image
curl -X PUT "https://api.livecaller.io/api/widgets/{widget-id}/features" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -F "features[0][name]=chat" \
  -F "features[0][settings][enabled]=true" \
  -F "features[0][image]=@/path/to/image.png"
```

---

## Best Practices

1. **Always validate on frontend** before sending to prevent unnecessary API calls
2. **Use debouncing** for auto-save features
3. **Handle 422 errors gracefully** and display field-specific errors
4. **Cache widget data** and only update changed sections
5. **Show loading states** during updates
6. **Implement optimistic updates** for better UX
7. **Use FormData** when uploading files (features endpoint)
8. **Sanitize user input** on frontend (backend also sanitizes)

---

## Changelog

### v2.0 (Current)
- New RESTful endpoints added
- Separate FormRequest validation per endpoint
- Improved field name derivation (PHP variable rules)
- Added label validation (must contain at least one letter)
- Input sanitization for name and field labels

### v1.0 (Legacy - Still Supported)
- Tab-based update endpoint
- Single monolithic update method

---

## Support

For questions or issues, contact the backend team or check the Laravel logs at `storage/logs/laravel.log`.

