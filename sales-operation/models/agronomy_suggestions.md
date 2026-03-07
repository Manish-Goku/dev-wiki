# agronomy_suggestions

**Collection**: `agronomy_suggestions` (in CRM-Database)
**Used by**: `agronomy/` module

## Schema
```javascript
{
  _id: ObjectId,
  lead_id: String,                          // refs leads.leadId
  agent_id: String,                         // refs agents.agentId or email
  crop_name: String,
  crop_area: String,
  crop_stage: String,                       // Vegetative|Panicle|Flowering|Fruiting|Mature Harvest
  irrigation_method: String,                // Flood|Drip|Sprinkler|Furrow
  crop_duration: String,
  issue: String,                            // crop problem description
  previous_used_products: [String],
  photo_urls: [String],                     // Firebase Storage URLs
  // Response (set when expert provides suggestion)
  is_suggested: Boolean,                    // false initially, true after response
  suggestion_description: String,
  products_sku: [String],                   // recommended product SKUs
  category: String,
  // timestamps
  created_at: DateTime,
  updated_at: DateTime
}
```

## Lookups
- `leads` -> `lead_first_name` (by lead_id)
- `products` -> `product_display_name` (by products_sku)

## File Upload
Firebase Storage: `suggestions_images/{lead_id}_{file_type}_{timestamp}_{uuid}.{ext}`

#active
