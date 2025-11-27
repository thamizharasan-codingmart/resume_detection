# Shareable Link Feature - Complete Schema Documentation

## Table of Contents

1. [Overview](#overview)
2. [Quick Reference](#quick-reference)
3. [Database Schema](#database-schema)
4. [Integration Guide](#integration-guide)
5. [Usage Examples](#usage-examples)
6. [Security Considerations](#security-considerations)
7. [API Endpoints](#api-endpoints)
8. [Frontend Components](#frontend-components)
9. [Migration Steps](#migration-steps)
10. [Verification Checklist](#verification-checklist)
11. [Testing Checklist](#testing-checklist)

---

## Overview

The Shareable Link feature allows users to create secure, shareable links for candidate profiles with customizable visibility settings. Users can select which candidate fields (resume, email, phone, etc.) are visible through the link, set optional password protection, and configure optional expiry dates. The system tracks all activities related to these links for analytics and security purposes.

### Link Format

The shareable links follow this format:
```
http://subdomain.quickrecruit.com/candidates/share/{shareToken}
```

Where `shareToken` is a unique, secure token generated for each shareable link.

---

## Quick Reference

### Database Tables

1. **candidate_shareable_links** - Stores shareable link configurations
2. **candidate_shareable_link_activities** - Tracks all link activities

### Key Features

✅ **Selectable Fields**: Choose which candidate fields are visible (resume, email, phone, etc.)  
✅ **Password Protection**: Optional password protection for links  
✅ **Expiry Date**: Optional expiry date/time for links  
✅ **Activity Tracking**: Track link opens, resume downloads, and security events  
✅ **Soft Delete**: Maintains audit trail with soft deletion  
✅ **Analytics**: IP address and user agent tracking for insights  

---

## Database Schema

### 1. CandidateShareableLink Model

This model stores the configuration and metadata for each shareable link.

#### Fields

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `id` | UUID | Primary key | Yes |
| `candidateId` | UUID | Reference to the candidate being shared | Yes |
| `organizationId` | UUID | Reference to the organization | Yes |
| `shareToken` | String (Unique) | Unique token used in the shareable URL | Yes |
| `passwordHash` | String (Nullable) | Hashed password if password protection is enabled | No |
| `expiresAt` | DateTime (Nullable) | Optional expiry date/time for the link | No |
| `isActive` | Boolean | Whether the link is currently active (default: true) | Yes |
| `visibleFields` | String[] | Array of field names that are visible through this link | Yes |
| `createdBy` | UUID | User who created the shareable link | Yes |
| `createdAt` | DateTime | Timestamp when the link was created | Yes |
| `updatedAt` | DateTime | Timestamp when the link was last updated | Yes |
| `deletedAt` | DateTime (Nullable) | Soft delete timestamp | No |

#### Relations

- **candidate**: Many-to-One relation with `Candidates` model
- **organization**: Many-to-One relation with `Organizations` model
- **createdByUser**: Many-to-One relation with `Users` model (via "ShareableLinkCreatedBy" relation)
- **activities**: One-to-Many relation with `CandidateShareableLinkActivity` model

#### Indexes

- `candidateId` - For quick lookups by candidate
- `organizationId` - For organization-level queries
- `shareToken` - For fast link resolution
- `isActive, deletedAt` - Composite index for active link queries

#### Visible Fields

The `visibleFields` array can contain any of the following field names (case-sensitive):
- `resume` - Candidate's resume file
- `email` - Candidate's email address
- `phone` - Candidate's phone number
- `name` - Candidate's name
- `experience` - Years of experience
- `skills` - Technical skills
- `education` - Educational background
- `workExperience` - Work history
- `certifications` - Professional certifications
- `languages` - Languages known
- `projects` - Project portfolio
- `linkedinUrl` - LinkedIn profile URL
- `githubUrl` - GitHub profile URL
- `portfolioUrl` - Portfolio URL
- `summary` - Professional summary
- `address` - Physical address
- `currentCompanyName` - Current employer
- `expectedCtc` - Expected salary
- `noticePeriodDuration` - Notice period

### 2. CandidateShareableLinkActivity Model

This model tracks all activities and interactions with shareable links for analytics and security monitoring.

#### Fields

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `id` | UUID | Primary key | Yes |
| `shareableLinkId` | UUID | Reference to the shareable link | Yes |
| `activityType` | Enum | Type of activity (see enum below) | Yes |
| `ipAddress` | String (Nullable) | IP address of the visitor | No |
| `userAgent` | String (Nullable) | Browser/device information | No |
| `metadata` | JSON (Nullable) | Additional metadata (flexible JSON field) | No |
| `createdAt` | DateTime | Timestamp when the activity occurred | Yes |

#### Relations

- **shareableLink**: Many-to-One relation with `CandidateShareableLink` model

#### Indexes

- `shareableLinkId` - For querying activities by link
- `activityType` - For filtering by activity type
- `createdAt` - For time-based queries and analytics

### 3. shareable_link_activity_type Enum

Enumeration of all possible activity types that can be tracked:

| Value | Description |
|-------|-------------|
| `LINK_OPENED` | When someone successfully opens the shareable link |
| `RESUME_DOWNLOADED` | When someone downloads the candidate's resume |
| `PASSWORD_VERIFIED` | When password is successfully verified |
| `PASSWORD_FAILED` | When password verification fails (for security monitoring) |
| `LINK_EXPIRED` | When link access is attempted after expiry date |
| `LINK_DEACTIVATED` | When link access is attempted but link is deactivated |

---

## Integration Guide

This section shows exactly where to add the new schema components in the main `schema.prisma` file.

### Step 1: Add Models After CandidateNotes

Add the following models after the `CandidateNotes` model (around line 841):

```prisma
// Add after CandidateNotes model (line 841)

model CandidateShareableLink {
  id                    String                              @id @default(uuid()) @db.Uuid
  candidateId           String                              @map("candidate_id") @db.Uuid
  candidate             Candidates                          @relation(fields: [candidateId], references: [id], onDelete: Cascade)
  organizationId        String                              @map("organization_id") @db.Uuid
  organization          Organizations                       @relation(fields: [organizationId], references: [id])
  shareToken            String                              @unique @map("share_token")
  passwordHash          String?                             @map("password_hash")
  expiresAt             DateTime?                           @map("expires_at")
  isActive              Boolean                              @default(true) @map("is_active")
  visibleFields         String[]                            @map("visible_fields")
  createdBy             String                              @map("created_by") @db.Uuid
  createdByUser         Users                               @relation("ShareableLinkCreatedBy", fields: [createdBy], references: [id])
  createdAt             DateTime                            @default(now()) @map("created_at")
  updatedAt             DateTime                            @updatedAt @map("updated_at")
  deletedAt             DateTime?                           @map("deleted_at")
  activities            CandidateShareableLinkActivity[]

  @@index([candidateId])
  @@index([organizationId])
  @@index([shareToken])
  @@index([isActive, deletedAt])
  @@map("candidate_shareable_links")
}

model CandidateShareableLinkActivity {
  id                    String                              @id @default(uuid()) @db.Uuid
  shareableLinkId       String                              @map("shareable_link_id") @db.Uuid
  shareableLink         CandidateShareableLink              @relation(fields: [shareableLinkId], references: [id], onDelete: Cascade)
  activityType          shareable_link_activity_type        @map("activity_type")
  ipAddress             String?                             @map("ip_address")
  userAgent             String?                             @map("user_agent")
  metadata              Json?
  createdAt             DateTime                            @default(now()) @map("created_at")

  @@index([shareableLinkId])
  @@index([activityType])
  @@index([createdAt])
  @@map("candidate_shareable_link_activities")
}
```

### Step 2: Add Enum

Add the enum after other enums (around line 4979, after `candidate_log_type` enum):

```prisma
// Add after candidate_log_type enum (around line 4979)

enum shareable_link_activity_type {
  LINK_OPENED
  RESUME_DOWNLOADED
  PASSWORD_VERIFIED
  PASSWORD_FAILED
  LINK_EXPIRED
  LINK_DEACTIVATED
}
```

### Step 3: Update Candidates Model

Add the relation to the `Candidates` model (around line 821, after `candidateNotes`):

```prisma
// In Candidates model, add after line 821:
  candidateNotes              CandidateNotes[]
  shareableLinks              CandidateShareableLink[]  // ADD THIS LINE
```

### Step 4: Update Users Model

Add the relation to the `Users` model. Find the section where relations are defined (around line 336, after `candidateNotesCreated`):

```prisma
// In Users model, add after candidateNotesCreated:
  candidateNotesCreated            CandidateNotes[]                   @relation("NoteCreatedBy")
  shareableLinksCreated            CandidateShareableLink[]           @relation("ShareableLinkCreatedBy")  // ADD THIS LINE
```

### Step 5: Update Organizations Model

Add the relation to the `Organizations` model. Find where other relations are defined (after line ~64):

```prisma
// In Organizations model, add:
  candidateShareableLinks         CandidateShareableLink[]
```

### Complete Integration Checklist

- [ ] Add `CandidateShareableLink` model after `CandidateNotes`
- [ ] Add `CandidateShareableLinkActivity` model after `CandidateShareableLink`
- [ ] Add `shareable_link_activity_type` enum
- [ ] Add `shareableLinks` relation to `Candidates` model
- [ ] Add `shareableLinksCreated` relation to `Users` model
- [ ] Add `candidateShareableLinks` relation to `Organizations` model
- [ ] Run `npx prisma format` to format the schema
- [ ] Run `npx prisma migrate dev --name add_shareable_link_feature` to create migration
- [ ] Run `npx prisma generate` to regenerate Prisma client

### Verification

After integration, verify the schema is correct:

```bash
npx prisma validate
```

If validation passes, you're ready to proceed with implementation!

---

## Usage Examples

### Creating a Shareable Link

```typescript
// Example: Create a shareable link with resume and email visible
const shareableLink = await prisma.candidateShareableLink.create({
  data: {
    candidateId: "candidate-uuid",
    organizationId: "org-uuid",
    shareToken: generateSecureToken(), // Generate unique token
    visibleFields: ["resume", "email", "name"],
    passwordHash: await hashPassword("secure-password"), // Optional
    expiresAt: new Date("2024-12-31"), // Optional
    isActive: true,
    createdBy: "user-uuid"
  }
});

// Generated link: http://subdomain.quickrecruit.com/candidates/share/{shareToken}
```

### Tracking Activities

```typescript
// Track when link is opened
await prisma.candidateShareableLinkActivity.create({
  data: {
    shareableLinkId: "link-uuid",
    activityType: "LINK_OPENED",
    ipAddress: request.ip,
    userAgent: request.headers["user-agent"],
    metadata: {
      referer: request.headers.referer,
      timestamp: new Date().toISOString()
    }
  }
});

// Track resume download
await prisma.candidateShareableLinkActivity.create({
  data: {
    shareableLinkId: "link-uuid",
    activityType: "RESUME_DOWNLOADED",
    ipAddress: request.ip,
    userAgent: request.headers["user-agent"]
  }
});
```

### Querying Analytics

```typescript
// Get all activities for a shareable link
const activities = await prisma.candidateShareableLinkActivity.findMany({
  where: {
    shareableLinkId: "link-uuid"
  },
  orderBy: {
    createdAt: "desc"
  }
});

// Count link opens
const openCount = await prisma.candidateShareableLinkActivity.count({
  where: {
    shareableLinkId: "link-uuid",
    activityType: "LINK_OPENED"
  }
});

// Count resume downloads
const downloadCount = await prisma.candidateShareableLinkActivity.count({
  where: {
    shareableLinkId: "link-uuid",
    activityType: "RESUME_DOWNLOADED"
  }
});
```

---

## Security Considerations

1. **Password Hashing**: Always hash passwords using a secure hashing algorithm (e.g., bcrypt) before storing in `passwordHash` field.

2. **Token Generation**: The `shareToken` should be:
   - Cryptographically secure and random
   - Long enough to prevent brute force attacks (recommended: 32+ characters)
   - URL-safe (no special characters that need encoding)

3. **Expiry Validation**: Always check `expiresAt` and `isActive` before allowing access to the link.

4. **IP Tracking**: Store IP addresses for security monitoring but be aware of privacy regulations (GDPR, etc.).

5. **Soft Delete**: Use `deletedAt` for soft deletion to maintain audit trails.

---

## API Endpoints

### Backend Endpoints (candidate-service)

1. **POST** `/candidates/:candidateId/shareable-links`
   - Create new shareable link
   - Body: `{ visibleFields, password?, expiresAt? }`
   - Response: Created link with shareToken

2. **GET** `/candidates/shareable-links/:linkId`
   - Get link details (for creator)
   - Response: Link configuration and analytics summary

3. **PUT** `/candidates/shareable-links/:linkId`
   - Update link (fields, password, expiry, active status)
   - Body: `{ visibleFields?, password?, expiresAt?, isActive? }`

4. **DELETE** `/candidates/shareable-links/:linkId`
   - Delete link (soft delete)
   - Sets `deletedAt` timestamp

5. **GET** `/candidates/share/:shareToken` (Public - No Auth Required)
   - View shared candidate profile
   - Requires password if `passwordHash` is set
   - Checks expiry and active status
   - Returns only selected `visibleFields`

6. **POST** `/candidates/share/:shareToken/verify-password` (Public)
   - Verify password for protected link
   - Body: `{ password }`
   - Response: Success/failure status

7. **GET** `/candidates/shareable-links/:linkId/activities`
   - Get analytics for a link
   - Returns activity counts and details
   - Query params: `?activityType=`, `?fromDate=`, `?toDate=`

---

## Frontend Components

### 1. Link Creation Form (app-dummy)

**Location**: Candidate profile page or dedicated shareable links section

**Features**:
- Field selection checkboxes (resume, email, phone, etc.)
- Password input (optional, with show/hide toggle)
- Expiry date picker (optional)
- Generate link button
- Copy link to clipboard functionality
- Preview of selected fields

### 2. Link Management Dashboard

**Location**: Candidate profile or settings section

**Features**:
- List all shareable links for a candidate
- Display link status (active/inactive/expired)
- Edit/Delete actions
- Copy link button
- View analytics button
- Quick toggle for active/inactive status

### 3. Public Share View

**Location**: Public route `/candidates/share/:shareToken`

**Features**:
- Password prompt modal (if password protected)
- Display selected candidate fields only
- Resume download button (if resume is in visibleFields)
- Expired link message
- Invalid link message
- Responsive design for mobile/desktop

### 4. Analytics Dashboard

**Location**: Link management section

**Features**:
- Link open count
- Resume download count
- Access timeline chart
- IP address tracking (if enabled)
- User agent information
- Date range filters
- Export analytics data

---

## Migration Steps

1. **Add Schema Components**: Follow the [Integration Guide](#integration-guide) to add models and enum to `schema.prisma`

2. **Format Schema**:
   ```bash
   npx prisma format
   ```

3. **Validate Schema**:
   ```bash
   npx prisma validate
   ```

4. **Create Migration**:
   ```bash
   npx prisma migrate dev --name add_shareable_link_feature
   ```

5. **Generate Prisma Client**:
   ```bash
   npx prisma generate
   ```

6. **Verify Migration**: Check that tables are created in database

---

## Verification Checklist

### Schema Design Review

- ✅ **Models Created**:
  - `CandidateShareableLink` - Stores link configuration
  - `CandidateShareableLinkActivity` - Tracks activities

- ✅ **Enum Created**:
  - `shareable_link_activity_type` - All activity types defined

- ✅ **Relations Added**:
  - `Candidates.shareableLinks`
  - `Users.shareableLinksCreated`
  - `Organizations.candidateShareableLinks`

### Feature Requirements Coverage

| Requirement | Status | Implementation |
|------------|--------|----------------|
| Select candidate fields (resume, email, phone) | ✅ | `visibleFields` array field |
| Create shareable link | ✅ | `CandidateShareableLink` model with `shareToken` |
| Password protection (optional) | ✅ | `passwordHash` field (nullable) |
| Expiry date (optional) | ✅ | `expiresAt` field (nullable) |
| Track link opens | ✅ | `LINK_OPENED` activity type |
| Track resume downloads | ✅ | `RESUME_DOWNLOADED` activity type |
| Link format: `/candidates/share/id` | ✅ | Uses `shareToken` in URL |
| Track activities | ✅ | `CandidateShareableLinkActivity` model |

### Database Design Best Practices

✅ **Indexes**: Proper indexes on frequently queried fields  
✅ **Soft Delete**: `deletedAt` field for audit trail  
✅ **Cascade Delete**: Activities deleted when link is deleted  
✅ **Unique Constraints**: `shareToken` is unique  
✅ **Nullable Fields**: Optional fields properly marked as nullable  
✅ **Timestamps**: `createdAt` and `updatedAt` for audit  
✅ **Relations**: Proper foreign key relationships  

### Performance Considerations

✅ **Indexes on**:
   - `candidateId` - Fast candidate lookups
   - `organizationId` - Organization-level queries
   - `shareToken` - Fast link resolution
   - `isActive, deletedAt` - Active link queries
   - `shareableLinkId` - Activity queries
   - `activityType` - Activity filtering
   - `createdAt` - Time-based analytics

---

## Testing Checklist

After implementation, test the following scenarios:

- [ ] Create shareable link without password
- [ ] Create shareable link with password
- [ ] Create shareable link with expiry
- [ ] Access link without password (should work)
- [ ] Access link with password (should prompt)
- [ ] Access expired link (should reject)
- [ ] Access deactivated link (should reject)
- [ ] Download resume (should track activity)
- [ ] View analytics (should show correct counts)
- [ ] Update link fields
- [ ] Update link password
- [ ] Update link expiry
- [ ] Toggle link active/inactive status
- [ ] Delete link (soft delete)
- [ ] Verify only selected fields are visible
- [ ] Verify password hashing is secure
- [ ] Verify token is unique and secure
- [ ] Test with multiple links for same candidate
- [ ] Test activity tracking accuracy
- [ ] Test IP address and user agent capture

---

## Notes

- The `shareToken` must be unique across all shareable links
- Password is optional - if `passwordHash` is null, no password is required
- Expiry is optional - if `expiresAt` is null, link never expires
- `isActive` flag allows disabling links without deleting them
- `metadata` field in activities table allows storing flexible additional data (e.g., download file size, browser type, etc.)
- All timestamps are in UTC
- Soft delete maintains data integrity and audit trails

---

## Next Steps

1. ✅ **Review Schema** - Verify all requirements are met
2. ⏳ **Integrate Schema** - Add to main `schema.prisma` file
3. ⏳ **Run Migration** - Create database tables
4. ⏳ **Implement Backend** - Create API endpoints
5. ⏳ **Implement Frontend** - Create UI components
6. ⏳ **Testing** - Test all functionality
7. ⏳ **Deploy** - Deploy to production

---

**Schema Status**: ✅ **READY FOR REVIEW**

All schema components are designed and documented. Once verified, proceed with integration and implementation.
