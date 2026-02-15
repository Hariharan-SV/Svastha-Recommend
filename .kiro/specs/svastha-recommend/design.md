# Design: Svastha-Recommend

## 1. System Architecture

### 1.1 High-Level Architecture

Svastha-Recommend follows an "Edge-Vault Shield" architecture with three primary layers:

```
┌─────────────────────────────────────────────────────────────┐
│                        User Device (Edge)                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  1. Medical Entity Extraction Layer (SLM)              │ │
│  │     - GLiNER / MedSpaCy models                         │ │
│  │     - Prescription OCR                                 │ │
│  └────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  2. PII Masking & Anonymization Layer                  │ │
│  │     - Redaction, Generalization, LDP                   │ │
│  └────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  3. Health Persona Mapping Layer                       │ │
│  │     - Public Taxonomy Bucketing                        │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                            ↓ (Masked Persona Only)
┌─────────────────────────────────────────────────────────────┐
│                    Cloud Services Layer                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Recommendation Engine                                  │ │
│  │  - ICMR-NIN / EFSA Guidelines Database                 │ │
│  │  - Product Catalog with Health Tags                    │ │
│  │  - Persona-to-Product Matching                         │ │
│  └────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  ONDC Integration Layer                                 │ │
│  │  - Buyer App API Integration                           │ │
│  │  - Search Interception                                 │ │
│  │  - Nudge Delivery                                      │ │
│  └────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Vendor Intelligence Layer                              │ │
│  │  - Anonymized Demand Aggregation                       │ │
│  │  - Pincode-level Forecasting                           │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                  External Integrations                       │
│  - ABDM/ABHA (Health Records)                               │
│  - ONDC Buyer Apps                                          │
│  - Insurance Providers (Optional)                           │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Component Breakdown

#### 1.2.1 Edge Medical Entity Extraction
- **Technology**: Small Language Models (GLiNER, MedSpaCy)
- **Deployment**: On-device (mobile CPU optimized)
- **Input**: Prescription images, PDF documents, ABDM health records
- **Output**: Structured health entities (conditions, medications, lab values)
- **Processing Time**: <3 seconds on mid-range devices

#### 1.2.2 PII Masking Layer
- **Redaction**: Remove names, exact addresses, hospital IDs
- **Generalization**: 
  - Ages → 5-year bins (e.g., 35 → "30-35")
  - Locations → 3-digit Pincodes (e.g., 560001 → "560")
- **Local Differential Privacy (LDP)**: Add controlled noise to health parameters
- **k-Anonymity**: Ensure at least k users share the same masked profile

#### 1.2.3 Health Persona Taxonomy
Public, transparent N-ary tree structure:

```
Root
├── Chronic Conditions
│   ├── Diabetes Management
│   │   ├── Type 1 - Low Glycemic
│   │   └── Type 2 - Moderate Carb
│   ├── Hypertension
│   │   ├── Low Sodium
│   │   └── DASH Diet
│   └── Cardiovascular
├── Nutritional Goals
│   ├── High Protein
│   ├── High Fiber
│   ├── Low Fat
│   └── Vitamin Deficiency
│       ├── Vitamin D
│       └── Iron
└── Dietary Restrictions
    ├── Gluten-Free
    ├── Lactose-Free
    └── Vegan/Vegetarian
```

**Persona Mapping Formula**:
```
Anonymity_Factor = N / M
where N = total population, M = number of persona buckets
```

## 2. Data Flow

### 2.1 User Onboarding Flow

```
1. User installs ONDC buyer app with Svastha extension
2. System displays "Notice and Consent" screen
   - Explains edge-only processing
   - Lists data usage purposes
   - Shows public persona taxonomy
3. User provides explicit consent
4. User links ABHA ID (optional) or uploads prescription
5. Edge entity extraction begins
6. PII masking applied immediately
7. Health persona assigned
8. User ready for personalized shopping
```

### 2.2 Shopping Recommendation Flow

```
1. User searches for product (e.g., "noodles") on ONDC app
2. Extension intercepts search query
3. Local persona retrieved from secure storage
4. Request sent to cloud: {persona_id, search_query, pincode}
5. Recommendation engine queries:
   - Product catalog with health tags
   - ICMR-NIN/EFSA guidelines for persona
6. Healthier alternatives identified
7. Nudge generated with guideline citation
8. Response returned to app
9. UI displays:
   - Original search results
   - Health nudge banner
   - Prioritized healthy alternatives
```

### 2.3 Vendor Demand Forecasting Flow

```
1. Weekly aggregation job runs
2. System groups masked personas by 3-digit Pincode
3. Calculates persona distribution (e.g., 30% High Vitamin D)
4. Identifies trending health needs
5. Generates demand forecast report
6. Vendor dashboard displays:
   - Top health personas in area
   - Recommended inventory adjustments
   - Product category trends
```

## 3. Technical Design

### 3.1 Edge Processing Module

**Technology Stack**:
- **Language**: Python (for ML models), JavaScript (for browser extension)
- **ML Framework**: ONNX Runtime for cross-platform model deployment
- **Models**: 
  - GLiNER (Generalist Named Entity Recognition)
  - MedSpaCy (Medical entity extraction)
  - Custom fine-tuned models for Indian prescriptions
- **OCR**: Tesseract OCR for prescription image processing

**Entity Extraction Pipeline**:
```python
# Pseudocode
def extract_entities(prescription_image):
    # Step 1: OCR
    text = ocr_engine.extract_text(prescription_image)
    
    # Step 2: Entity extraction
    entities = slm_model.extract({
        'conditions': [],
        'medications': [],
        'lab_values': [],
        'dietary_restrictions': []
    })
    
    # Step 3: Immediate PII masking
    masked_entities = apply_pii_masking(entities)
    
    # Step 4: Delete original image
    delete_file(prescription_image)
    
    return masked_entities
```

### 3.2 PII Masking Implementation

**Masking Techniques**:
```python
# Pseudocode
def apply_pii_masking(entities):
    masked = {}
    
    # Redaction
    masked['name'] = None  # Remove completely
    masked['hospital_id'] = None
    
    # Generalization
    masked['age_bin'] = generalize_age(entities['age'])  # 35 → "30-35"
    masked['pincode_prefix'] = entities['pincode'][:3]   # 560001 → "560"
    
    # Local Differential Privacy
    for param in ['blood_sugar', 'blood_pressure']:
        masked[param] = add_laplace_noise(
            entities[param], 
            epsilon=1.0  # Privacy budget
        )
    
    return masked
```

**k-Anonymity Validation**:
```python
def validate_k_anonymity(masked_profile, k=5):
    # Query local database for similar profiles
    similar_count = count_similar_profiles(masked_profile)
    
    if similar_count < k:
        # Generalize further or suppress
        return generalize_more(masked_profile)
    
    return masked_profile
```

### 3.3 Health Persona Mapping

**Persona Assignment Algorithm**:
```python
def assign_persona(masked_entities):
    persona_scores = {}
    
    # Score against each persona in taxonomy
    for persona in PERSONA_TAXONOMY:
        score = calculate_match_score(
            masked_entities,
            persona.criteria
        )
        persona_scores[persona.id] = score
    
    # Assign to best matching persona
    best_persona = max(persona_scores, key=persona_scores.get)
    
    return {
        'persona_id': best_persona,
        'confidence': persona_scores[best_persona],
        'timestamp': current_timestamp()
    }
```

### 3.4 Recommendation Engine

**Guideline Database Schema**:
```json
{
  "guideline_id": "ICMR-NIN-2024-SODIUM",
  "source": "ICMR-NIN 2024",
  "parameter": "sodium",
  "limit": {
    "value": 5,
    "unit": "g/day"
  },
  "applicable_personas": [
    "hypertension-low-sodium",
    "cardiovascular-dash-diet"
  ],
  "recommendation_text": "NIN Guidelines recommend limiting salt intake to <5g/day"
}
```

**Product Tagging Schema**:
```json
{
  "product_id": "ONDC-PROD-12345",
  "name": "Whole Grain Ragi Noodles",
  "health_tags": [
    "whole-grain",
    "low-sodium",
    "high-fiber",
    "gluten-free"
  ],
  "nutritional_info": {
    "sodium_per_100g": 120,
    "fiber_per_100g": 8,
    "glycemic_index": "low"
  },
  "guideline_compliance": [
    "ICMR-NIN-2024-WHOLE-GRAIN",
    "ICMR-NIN-2024-SODIUM"
  ]
}
```

**Recommendation Algorithm**:
```python
def generate_recommendations(persona_id, search_query, pincode):
    # Step 1: Get applicable guidelines
    guidelines = get_guidelines_for_persona(persona_id)
    
    # Step 2: Search products
    products = search_product_catalog(search_query, pincode)
    
    # Step 3: Score products against guidelines
    scored_products = []
    for product in products:
        score = 0
        for guideline in guidelines:
            if guideline.id in product.guideline_compliance:
                score += guideline.weight
        scored_products.append((product, score))
    
    # Step 4: Sort by health score
    scored_products.sort(key=lambda x: x[1], reverse=True)
    
    # Step 5: Generate nudge
    nudge = generate_nudge_text(guidelines, scored_products[0])
    
    return {
        'recommended_products': scored_products[:10],
        'nudge': nudge,
        'guideline_citations': [g.citation for g in guidelines]
    }
```

### 3.5 ONDC Integration

**Browser Extension Architecture**:
```javascript
// Content script injected into ONDC buyer app
class SvasthaExtension {
    constructor() {
        this.personaId = this.loadPersonaFromStorage();
        this.interceptSearches();
    }
    
    interceptSearches() {
        // Hook into search input
        document.querySelector('#search-box').addEventListener('input', 
            debounce(this.handleSearch.bind(this), 500)
        );
    }
    
    async handleSearch(event) {
        const query = event.target.value;
        const pincode = this.getUserPincode();
        
        // Call recommendation API
        const recommendations = await this.fetchRecommendations({
            persona_id: this.personaId,
            query: query,
            pincode: pincode
        });
        
        // Inject nudge into UI
        this.displayNudge(recommendations.nudge);
        this.reorderProducts(recommendations.recommended_products);
    }
    
    displayNudge(nudge) {
        const banner = document.createElement('div');
        banner.className = 'svastha-nudge';
        banner.innerHTML = `
            <div class="nudge-icon">💚</div>
            <div class="nudge-text">${nudge.text}</div>
            <div class="nudge-source">${nudge.citation}</div>
        `;
        document.querySelector('#search-results').prepend(banner);
    }
}
```

### 3.6 Vendor Dashboard

**Demand Aggregation Service**:
```python
def generate_vendor_forecast(vendor_pincode_prefix):
    # Step 1: Get all personas in area (anonymized)
    personas_in_area = aggregate_personas_by_pincode(vendor_pincode_prefix)
    
    # Step 2: Calculate distribution
    distribution = {}
    total = len(personas_in_area)
    for persona in set(personas_in_area):
        count = personas_in_area.count(persona)
        distribution[persona] = (count / total) * 100
    
    # Step 3: Map to product categories
    recommended_categories = []
    for persona, percentage in distribution.items():
        if percentage > 10:  # Significant demand
            categories = get_product_categories_for_persona(persona)
            recommended_categories.extend(categories)
    
    # Step 4: Generate forecast
    return {
        'pincode_prefix': vendor_pincode_prefix,
        'persona_distribution': distribution,
        'recommended_categories': Counter(recommended_categories).most_common(10),
        'trending_health_needs': identify_trends(personas_in_area),
        'forecast_period': 'next_7_days'
    }
```

## 4. Security Design

### 4.1 Data Encryption
- **At Rest**: AES-256 encryption for local persona storage
- **In Transit**: TLS 1.3 for all API communications
- **Key Management**: Device-specific keys stored in secure enclave

### 4.2 Authentication & Authorization
- **User Authentication**: OAuth 2.0 with ABDM
- **API Authentication**: JWT tokens with short expiry (15 minutes)
- **Vendor Authentication**: Separate vendor portal with role-based access

### 4.3 Audit Logging
- All persona assignments logged (without PII)
- All recommendation requests logged
- Regular compliance audits performed
- Logs retained for 90 days

## 5. Compliance Design

### 5.1 DPDP Act 2023 Compliance
- **Data Fiduciary Role**: System acts as fiduciary with edge-first architecture
- **Consent Management**: Explicit opt-in with clear purpose specification
- **Data Minimization**: Only necessary health entities extracted
- **Right to Erasure**: Users can delete persona and all associated data

### 5.2 GDPR Compliance
- **Privacy by Design**: Edge processing ensures no cloud storage of health data
- **Data Subject Rights**: Users can access, rectify, and delete their data
- **Data Protection Impact Assessment (DPIA)**: Conducted for high-risk processing
- **DPO Appointment**: Data Protection Officer designated

### 5.3 EU AI Act Compliance
- **Transparency**: Clear AI disclosure in UI
- **Risk Classification**: System classified as "Limited Risk"
- **Verified Data**: All guidelines from ICMR-NIN and EFSA
- **Public Taxonomy**: Persona buckets publicly documented

## 6. User Interface Design

### 6.1 Consent Screen
```
┌─────────────────────────────────────────────┐
│  Welcome to Svastha-Recommend               │
│                                             │
│  🔒 Your Privacy, Your Control              │
│                                             │
│  How it works:                              │
│  ✓ Your health data stays on YOUR device   │
│  ✓ We only use anonymous health profiles   │
│  ✓ Get personalized health recommendations │
│                                             │
│  What we collect:                           │
│  • Health conditions (anonymized)           │
│  • Dietary needs (anonymized)               │
│  • Shopping preferences                     │
│                                             │
│  [View Public Persona Taxonomy]             │
│  [Read Privacy Policy]                      │
│                                             │
│  [ ] I consent to edge-only processing      │
│                                             │
│  [Continue]  [Learn More]                   │
└─────────────────────────────────────────────┘
```

### 6.2 Health Nudge Display
```
┌─────────────────────────────────────────────┐
│  Search: "noodles"                          │
├─────────────────────────────────────────────┤
│  💚 Health Tip                              │
│  NIN Guidelines recommend 50% of cereal     │
│  intake from whole grains and millets.      │
│  Consider these healthier alternatives:     │
│  [View Guidelines]                          │
├─────────────────────────────────────────────┤
│  ⭐ Recommended for You                     │
│  🌾 Whole Grain Ragi Noodles               │
│  ✓ Low Sodium  ✓ High Fiber                │
│  ₹45  [Add to Cart]                         │
├─────────────────────────────────────────────┤
│  Regular Noodles                            │
│  ₹30  [Add to Cart]                         │
└─────────────────────────────────────────────┘
```

### 6.3 Vendor Dashboard
```
┌─────────────────────────────────────────────┐
│  Demand Forecast - Pincode 560              │
│  Week of Feb 15-22, 2026                    │
├─────────────────────────────────────────────┤
│  Health Trends in Your Area:                │
│  📊 30% High Vitamin D needs                │
│  📊 25% Low Sodium preferences              │
│  📊 20% High Fiber goals                    │
│                                             │
│  Recommended Inventory:                     │
│  🥛 Fortified Milk (+15% demand)            │
│  🥚 Eggs (+12% demand)                      │
│  🌾 Whole Grain Products (+10% demand)      │
│                                             │
│  [View Detailed Report]                     │
│  [Optimize Inventory]                       │
└─────────────────────────────────────────────┘
```

## 7. Testing Strategy

### 7.1 Unit Testing
- Test entity extraction accuracy (>90% precision)
- Test PII masking completeness (100% PII removed)
- Test persona assignment consistency
- Test recommendation algorithm correctness

### 7.2 Integration Testing
- Test ABDM integration for health record retrieval
- Test ONDC API integration for product search
- Test end-to-end flow from prescription upload to recommendation

### 7.3 Security Testing
- Penetration testing for API endpoints
- Privacy audit for data leakage
- Compliance validation against DPDP/GDPR requirements

### 7.4 Performance Testing
- Load testing for concurrent users (target: 1M users)
- Edge processing latency testing (target: <3s)
- Recommendation API response time (target: <1s)

### 7.5 User Acceptance Testing
- Test with real users for usability
- Validate nudge effectiveness (behavior change metrics)
- Gather feedback on transparency and trust

## 8. Deployment Strategy

### 8.1 Phased Rollout
1. **Phase 1**: Pilot with 1,000 users in Bangalore (Pincode 560)
2. **Phase 2**: Expand to 10,000 users across Karnataka
3. **Phase 3**: National rollout across India
4. **Phase 4**: International expansion (EU market)

### 8.2 Infrastructure
- **Edge**: Browser extension + mobile SDK
- **Cloud**: AWS/Azure with India region for data residency
- **CDN**: CloudFlare for static assets
- **Database**: PostgreSQL for product catalog, Redis for caching

### 8.3 Monitoring
- **Application Monitoring**: New Relic / Datadog
- **Error Tracking**: Sentry
- **Analytics**: Mixpanel for user behavior (privacy-compliant)
- **Compliance Monitoring**: Automated DPDP/GDPR compliance checks

## 9. Future Enhancements

### 9.1 Advanced Features
- Multi-language support for regional Indian languages
- Voice-based prescription input
- Integration with fitness trackers for holistic health profiles
- AI-powered meal planning based on shopping history

### 9.2 Expanded Integrations
- Integration with telemedicine platforms
- Partnership with health insurance providers
- Integration with pharmacy chains for medication reminders
- Collaboration with government health programs

### 9.3 Research Opportunities
- Federated learning for improving entity extraction models
- Differential privacy research for better anonymization
- Behavioral economics research on nudge effectiveness
- Public health impact studies

## 10. Success Metrics

### 10.1 Privacy Metrics
- Zero cloud storage of raw medical data (100% compliance)
- k-anonymity factor > 5 for all personas
- Zero data breaches or privacy incidents

### 10.2 User Metrics
- User adoption rate (target: 100K users in 6 months)
- Consent rate (target: >70%)
- User retention (target: >60% monthly active)
- User satisfaction score (target: >4/5)

### 10.3 Health Impact Metrics
- Percentage of users making healthier choices (target: >40%)
- Reduction in high-sodium product purchases (target: 20%)
- Increase in whole grain product purchases (target: 30%)

### 10.4 Business Metrics
- Vendor adoption rate (target: 1,000 vendors in 6 months)
- Inventory optimization impact (target: 15% waste reduction)
- Revenue from premium features (target: ₹10L/month)

## 11. Risk Mitigation

### 11.1 Technical Risks
- **Risk**: Edge models fail on low-end devices
- **Mitigation**: Provide cloud fallback with explicit consent

### 11.2 Regulatory Risks
- **Risk**: Changes in DPDP/GDPR regulations
- **Mitigation**: Modular compliance layer for quick updates

### 11.3 User Adoption Risks
- **Risk**: Users don't trust AI recommendations
- **Mitigation**: Transparent taxonomy, guideline citations, user education

### 11.4 Business Risks
- **Risk**: ONDC integration challenges
- **Mitigation**: Build standalone app as backup distribution channel
