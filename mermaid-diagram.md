```mermaid
flowchart TB
    consumer["Consumer App"]
    marketplace["Marketplace"]
    pricing["Pricing API"]
    merchant["Merchant Service<br/>(Through Merchant Caching)"]
    mongo[(Mongo DB)]
    cnp["CNP Service"]
    sales[(Sales Details Table<br/>(Subscriber DB))]

    consumer -->|1. Create Carts with Product & Select payment Method| marketplace
    marketplace -->|7. Display Transaction Fee Amount| consumer

    marketplace -->|2. Request Fee Details| pricing
    pricing -->|6. Compute Fee amounts| marketplace

    pricing -->|3. Request Fee Rates based on payment method type & merchant Id| merchant
    merchant -->|5. Send respective fees| pricing
    mongo -->|4. Send stored fee rates in merchant collection| merchant

    marketplace -->|8. Send Payment| cnp
    cnp -->|9. Payment confirmation| marketplace

    marketplace -->|10. Store Platform Fee after checkout| sales
```
