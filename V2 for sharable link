# Shareable Link V2 - Template Feature Schema Requirements

## V2 Overview

V2 introduces **reusable templates** that allow users to:
- Create template configurations with sections and fields
- Save templates for future use
- Apply templates when creating shareable links
- Still have option to create custom links (without template)

---

## Schema Additions Required for V2

### 1. ShareableLinkTemplate Model

**Purpose**: Store reusable template configurations

**Fields Needed**:
- `id` (UUID) - Primary key
- `name` (String) - Template name (e.g., "Basic Candidate Info", "Full Profile")
- `description` (String, nullable) - Optional template description
- `organizationId` (UUID) - Organization that owns the template
- `createdBy` (UUID) - User who created the template
- `isDefault` (Boolean) - Whether this is a default/system template
- `isPublic` (Boolean) - Whether template is shared across organization
- `createdAt` (DateTime)
- `updatedAt` (DateTime)
- `deletedAt` (DateTime, nullable) - Soft delete

**Relations**:
- `organization` → Organizations
- `createdByUser` → Users
- `templateSections` → shareableLinkTemplateSections[]
- `shareableLinks` → CandidateShareableLink[] (links using this template)

**Indexes**:
- `organizationId`
- `createdBy`
- `isDefault`
- `isPublic, deletedAt`

---

### 2. shareableLinkTemplateSections Model

**Purpose**: Store sections within a template (similar to shareableLinkSections but for templates)

**Fields Needed**:
- `id` (UUID) - Primary key
- `templateId` (UUID) - Reference to template
- `sectionName` (String) - Section name
- `sectionKey` (String, nullable) - Optional section key
- `order` (Int) - Display order
- `isVisible` (Boolean) - Default visibility
- `createdAt` (DateTime)
- `updatedAt` (DateTime)
- `deletedAt` (DateTime, nullable)

**Relations**:
- `template` → ShareableLinkTemplate
- `templateFields` → shareableLinkTemplateFields[]

**Indexes**:
- `templateId`
- `templateId, order`
- Unique constraint: `[templateId, sectionKey]` (if sectionKey is provided)

---

### 3. shareableLinkTemplateFields Model

**Purpose**: Store fields within template sections

**Fields Needed**:
- `id` (UUID) - Primary key
- `templateSectionId` (UUID) - Reference to template section
- `fieldName` (String) - Field name
- `fieldPath` (String, nullable) - Optional field path
- `dataType` (String, nullable) - Optional data type
- `order` (Int) - Display order
- `isVisible` (Boolean) - Default visibility
- `createdAt` (DateTime)
- `updatedAt` (DateTime)
- `deletedAt` (DateTime, nullable)

**Relations**:
- `templateSection` → shareableLinkTemplateSections

**Indexes**:
- `templateSectionId`
- `templateSectionId, order`
- Unique constraint: `[templateSectionId, fieldName]`

---

### 4. Update CandidateShareableLink Model

**Additional Fields Needed**:
- `templateId` (UUID, nullable) - Reference to template used (if any)
- `isFromTemplate` (Boolean) - Whether link was created from template
- `templateAppliedAt` (DateTime, nullable) - When template was applied

**Additional Relations**:
- `template` → ShareableLinkTemplate (nullable)

**Additional Indexes**:
- `templateId`

**Note**: 
- If `templateId` is null, link is custom (V1 behavior)
- If `templateId` is set, link uses template configuration
- User can still override template sections/fields when creating link

---

## Schema Structure for V2

```
ShareableLinkTemplate (1)
  ├── shareableLinkTemplateSections (N)
  │   └── shareableLinkTemplateFields (N)
  └── shareableLinks (N) → CandidateShareableLink

CandidateShareableLink (1)
  ├── template (optional) → ShareableLinkTemplate
  ├── shareableLinkCandidates (N)
  ├── shareableLinkSections (N)  // Can be from template or custom
  │   └── shareableLinkFields (N)
  └── activities (N)
```

---

## Key Design Decisions

### 1. Template vs Custom Links

**Option A: Template creates sections/fields on link creation**
- When template is applied, copy template sections/fields to `shareableLinkSections` and `shareableLinkFields`
- Link is independent after creation
- Template changes don't affect existing links

**Option B: Link references template (live reference)**
- Link stores `templateId` reference
- Sections/fields are read from template at runtime
- Template changes affect all links using it
- More flexible but less control per link

**Recommendation**: **Option A** (copy on creation)
- Better for V1 compatibility
- Links remain independent
- Users can still customize after applying template

### 2. Default Templates

- System-wide default templates (`isDefault: true`)
- Organization-specific templates
- User-created templates

### 3. Template Sharing

- `isPublic: true` - Template visible to all users in organization
- `isPublic: false` - Template only visible to creator
- Future: Share templates across organizations

---

## Migration Path from V1 to V2

1. **Add new template models** (non-breaking)
2. **Add optional fields to CandidateShareableLink** (backward compatible)
3. **Existing V1 links continue to work** (templateId = null)
4. **New V2 links can use templates** (templateId set)

---

## Additional Considerations

### Template Versioning (Future Enhancement)
- Track template versions
- Allow reverting to previous template version
- Track which version was used for each link

### Template Categories/Tags (Future Enhancement)
- Categorize templates (e.g., "Basic", "Detailed", "Executive")
- Search/filter templates by category
- Tags for better organization

### Template Preview
- Preview template before applying
- Show sample data with template configuration
- Compare templates side-by-side

### Template Analytics
- Track template usage (how many links use each template)
- Most popular templates
- Template performance metrics

---

## Summary of Schema Additions

### New Models (3):
1. ✅ `ShareableLinkTemplate` - Template storage
2. ✅ `shareableLinkTemplateSections` - Template sections
3. ✅ `shareableLinkTemplateFields` - Template fields

### Updated Models (1):
1. ✅ `CandidateShareableLink` - Add `templateId`, `isFromTemplate`, `templateAppliedAt`

### New Relations:
- `ShareableLinkTemplate` → `Organizations`
- `ShareableLinkTemplate` → `Users` (createdBy)
- `ShareableLinkTemplate` → `CandidateShareableLink[]`
- `shareableLinkTemplateSections` → `ShareableLinkTemplate`
- `shareableLinkTemplateFields` → `shareableLinkTemplateSections`
- `CandidateShareableLink` → `ShareableLinkTemplate` (optional)

### New Indexes:
- Template lookup by organization
- Template lookup by creator
- Default templates
- Public templates
- Links by template

---

## Implementation Notes

1. **Backward Compatibility**: V1 links (without template) continue to work
2. **Flexibility**: Users can still create custom links even with templates available
3. **Template Management**: CRUD operations for templates
4. **Template Application**: When creating link, user can:
   - Select a template (applies template sections/fields)
   - Create custom (no template)
   - Start from template and customize

---

## V2 Feature Flow

1. **Template Creation**:
   - User creates template with sections/fields
   - Saves template with name/description
   - Template available for future use

2. **Link Creation with Template**:
   - User selects candidates
   - User selects template (or creates custom)
   - If template selected:
     - Template sections/fields copied to link
     - User can still customize
   - Configure password/expiry
   - Preview and create

3. **Template Management**:
   - View all templates
   - Edit templates
   - Delete templates (soft delete)
   - Duplicate templates
   - Set as default

---

**Status**: Schema requirements documented - Ready for V2 implementation when needed

