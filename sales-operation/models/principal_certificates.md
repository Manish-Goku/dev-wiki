# principal_certificates

**Collection**: `principal_certificates` (in CRM-Database)
**Used by**: `agronomy/` module

## Schema
```javascript
{
  _id: ObjectId,
  email: String,                            // agent email (EmailStr)
  agent_name: String,
  customer_name: String,
  customer_shop_name: String,
  customer_contact_number: String,          // 10 digits
  customer_email_id: String,                // EmailStr
  approval_from_floor_manager: String,
  state: String,
  pincode: String,                          // 6 digits
  shop_full_address: String,
  gst_number: String,                       // 15 chars
  // Document URLs (S3)
  gst_certificate_url: String,
  pesticide_license_url: String,
  fertilizer_license_url: String,
  aadhaar_front_url: String,
  aadhaar_back_url: String,
  pan_card_url: String,
  // Business info
  annual_business_commitment: String,
  doing_business_from: String,
  app_downloaded: String,
  kyc_done_on_app: String,
  ready_for_business_commitment: String,
  // Computed from orders collection
  total_orders_till_now: Number,
  total_order_value_till_now: Number,
  total_months_from_first_order: Number,
  // Metadata
  created_at: DateTime,
  created_by: String                        // agent email
}
```

## File Upload
S3 path: `agronomy/registrations/{customer_email}/{field_name}_{timestamp}.{ext}`

#active
