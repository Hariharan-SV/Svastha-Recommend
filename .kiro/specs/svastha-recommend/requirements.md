# Requirements: Svastha-Recommend

## 1. Overview

Svastha-Recommend is a privacy-preserving health-integrated e-commerce recommendation system for the "AI for Bharat" initiative. The system processes medical data locally on the user's device using edge computing, ensuring no sensitive health information is stored in the cloud. It provides personalized shopping recommendations based on health profiles while maintaining strict privacy compliance with DPDP Act 2023 and GDPR.

## 2. User Stories

### 2.1 End User Stories

**US-1: Health-Aware Shopping**
As a user with health conditions, I want to receive personalized product recommendations based on my medical records, so that I can make healthier purchasing decisions without compromising my privacy.

**US-2: Prescription Processing**
As a user, I want to upload my prescription or medical records, so that the system can extract relevant health information locally on my device without sending it to the cloud.

**US-3: Privacy Control**
As a privacy-conscious user, I want my medical data to be processed only on my device, so that I have full control over my sensitive health information.

**US-4: Transparent Recommendations**
As a user, I want to understand why certain products are recommended to me based on verified dietary guidelines, so that I can trust the system's suggestions.

### 2.2 Vendor Stories

**US-5: Demand Forecasting**
As a local vendor, I want to receive anonymized demand forecasts for health-related products in my area, so that I can optimize my inventory and reduce waste.

**US-6: Product Visibility**
As a vendor, I want my health-compliant products to be prioritized for users with relevant health needs, so that I can increase sales of nutritious items.

### 2.3 System Integration Stories

**US-7: ONDC Integration**
As a system integrator, I want Svastha-Recommend to work seamlessly with ONDC buyer apps, so that health nudges appear at the point of purchase.

**US-8: ABDM Integration**
As a healthcare system administrator, I want the system to integrate with ABDM as a Health Information User (HIU), so that users can access their health records via ABHA ID.

## 3. Acceptance Criteria

### 3.1 Privacy and Security

**AC-1.1: Edge-Only Processing**
- Medical data MUST be processed exclusively on the user's device
- No raw medical records SHALL be transmitted to cloud servers
- All health entity extraction MUST occur locally using on-device models

**AC-1.2: PII Masking**
- Personal identifiable information (names, addresses, exact ages) MUST be redacted before any processing
- Ages MUST be generalized into 5-year bins
- Locations MUST be generalized to 3-digit Pincodes
- Local Differential Privacy (LDP) MUST be applied to health parameters

**AC-1.3: Data Residency**
- Medical data MUST remain on the user's device at all times
- Only masked, anonymized health personas MAY be used for recommendations
- Users MUST explicitly consent before any data export

### 3.2 Medical Entity Extraction

**AC-2.1: Local NLP Processing**
- System MUST use Small Language Models (SLMs) like GLiNER or MedSpaCy for entity extraction
- Models MUST run efficiently on mobile CPUs
- Extraction MUST identify conditions (e.g., "Type 2 Diabetes"), medications (e.g., "Metformin 500mg"), and dietary restrictions

**AC-2.2: Prescription Upload**
- Users MUST be able to upload prescriptions via photo or PDF
- System MUST extract relevant clinical entities from uploaded documents
- Original documents MUST be deleted after entity extraction

### 3.3 Recommendation Engine

**AC-3.1: Verified Guidelines Integration**
- All recommendations MUST be mapped to ICMR-NIN Dietary Guidelines for Indians 2024 or EFSA Dietary Reference Values
- Sodium recommendations MUST align with <5g/day limit
- Sugar recommendations MUST align with <5% of total calories limit
- Whole grain recommendations MUST prioritize 50% cereal intake from whole grains and millets

**AC-3.2: Transparent Persona Bucketing**
- System MUST use a public, transparent taxonomy for health personas
- Persona buckets (e.g., "High Fiber/Low Glycemic preference") MUST be publicly documented
- Recommendation logic MUST be auditable

**AC-3.3: Shopping Nudges**
- System MUST intercept product searches on ONDC buyer apps
- Recommendations MUST prioritize healthier alternatives based on user's health profile
- Nudges MUST use suggestive language (e.g., "Boost your energy with X") rather than restrictive commands

### 3.4 Regulatory Compliance

**AC-4.1: DPDP Act 2023 Compliance**
- System MUST implement "Notice and Consent" architecture
- Consent MUST be free, specific, informed, and unconditional
- System MUST act as a Data Fiduciary with edge-first architecture

**AC-4.2: GDPR Compliance**
- System MUST comply with GDPR Article 9 for special category health data
- System MUST implement "Privacy by Design" (Article 25)
- System MUST obtain explicit consent for health data processing

**AC-4.3: EU AI Act Compliance**
- System MUST be transparent about AI interaction
- System MUST avoid subliminal manipulation
- System MUST use verified data sources (EFSA/ICMR-NIN)

### 3.5 User Experience

**AC-5.1: Consent Management**
- Users MUST explicitly opt-in to use health records for recommendations
- Users MUST be able to revoke consent at any time
- System MUST clearly explain data usage before consent

**AC-5.2: Transparency**
- Users MUST be informed they are interacting with an AI system
- Recommendations MUST display the guideline source (e.g., "NIN Guidelines recommend...")
- Users MUST be able to view their assigned health persona

**AC-5.3: Bias Mitigation**
- Healthy recommendations MUST be available across all price brackets
- System MUST avoid "health shaming" language
- Algorithmic fairness audits MUST be performed regularly

### 3.6 Vendor Features

**AC-6.1: Demand Forecasting**
- Vendors MUST receive weekly anonymized demand forecasts for their Pincode
- Forecasts MUST aggregate masked persona types in the area
- No individual user data SHALL be exposed to vendors

**AC-6.2: Inventory Optimization**
- Vendors MUST be able to view trending health-related product categories
- System MUST provide actionable insights for inventory management

### 3.7 Optional Features

**AC-7.1: Insurance Integration** (Optional)
- Users MAY opt-in to share their "Svastha Score" with health insurers
- Score MUST be calculated locally
- Only aggregate score MAY be shared via encrypted ONDC gateway
- Integration MUST comply with IRDAI 2024 circular

## 4. Non-Functional Requirements

### 4.1 Performance
- Entity extraction MUST complete within 3 seconds on mid-range mobile devices
- Recommendation generation MUST complete within 1 second
- System MUST support offline processing for entity extraction

### 4.2 Scalability
- System MUST support millions of concurrent users
- Edge architecture MUST minimize server load
- Persona taxonomy MUST scale to accommodate diverse health profiles

### 4.3 Security
- All data transmission MUST use end-to-end encryption
- System MUST implement secure key management for user data
- Regular security audits MUST be conducted

### 4.4 Reliability
- System MUST have 99.9% uptime for recommendation services
- Edge processing MUST have fallback mechanisms for model failures
- Data integrity MUST be maintained throughout the processing pipeline

### 4.5 Maintainability
- Code MUST follow industry best practices
- System MUST have comprehensive documentation
- Models MUST be updatable without breaking existing functionality

## 5. Constraints

### 5.1 Technical Constraints
- Medical data processing MUST occur on-device only
- System MUST use Small Language Models optimized for mobile CPUs
- Integration MUST work within ONDC and ABDM frameworks

### 5.2 Regulatory Constraints
- System MUST comply with DPDP Act 2023, GDPR, and EU AI Act
- Dietary recommendations MUST align with ICMR-NIN 2024 and EFSA guidelines
- Data processing MUST follow Indian data residency requirements

### 5.3 Business Constraints
- System MUST integrate with existing ONDC buyer apps
- Vendor features MUST not expose individual user data
- Solution MUST be cost-effective for deployment at scale

## 6. Assumptions

- Users have smartphones capable of running on-device ML models
- Users have access to their medical records via ABDM/ABHA
- ONDC buyer apps provide APIs for recommendation integration
- Verified dietary guidelines (ICMR-NIN, EFSA) are publicly accessible
- Users are willing to upload prescriptions for personalized recommendations

## 7. Dependencies

- ABDM infrastructure for health record access
- ONDC platform for e-commerce integration
- ICMR-NIN Dietary Guidelines for Indians 2024
- EFSA Dietary Reference Values
- Small Language Models (GLiNER, MedSpaCy) for entity extraction
- Mobile device capabilities for on-device processing
