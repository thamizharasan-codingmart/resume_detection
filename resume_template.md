# Resume Template Implementation Documentation

## API Endpoint: `get-available-template`

### Endpoint Details
- **Path**: `/resume-builder/get-available-template`
- **Method**: `GET`
- **Service**: `billingusers-service`
- **Route File**: `billingusers-service/src/api/resume-builder/resumeBuilder.routes.ts`
- **Controller**: `billingusers-service/src/api/resume-builder/resumeBuilder.controller.ts`
- **DAO**: `billingusers-service/src/api/resume-builder/resumeBuilder.dao.ts`

### How It Works

#### 1. Controller Method (`getAvailableTemplate`)
```typescript
// Location: resumeBuilder.controller.ts:228-233
public getAvailableTemplate = async (request: FastifyRequest, reply: FastifyReply) => {
    const templateDetails = await resumeBuilderDao.getAvailableTemplate();
    return reply
        .status(200)
        .send(fmt.formatResponse(templateDetails, 'Successfully fetched available template'));
};
```

#### 2. DAO Method (`getAvailableTemplate`)
```typescript
// Location: resumeBuilder.dao.ts:447-458
public getAvailableTemplate = async () => {
    // Step 1: Find the default template (isDefault=true, organizationId=null)
    const defaultTemplate = await client.resumeTemplates.findFirst({
        where: {
            isDefault: true,
            organizationId: null,
            deletedAt: null,
        },
    });
    
    // Step 2: If no default template found, return null
    if (!defaultTemplate) return null;
    
    // Step 3: Get template with inheritance (includes sections and fields)
    const templateDetails = await this.getTemplateWithInheritance(defaultTemplate.id);
    return templateDetails;
};
```

#### 3. Template with Inheritance (`getTemplateWithInheritance`)
This method builds a complete template structure by:
- Getting the base template
- Fetching current template's sections and fields
- Fetching parent template's sections (if `parentTemplateId` exists)
- Merging sections and fields with inheritance logic:
  - Current template sections/fields take precedence
  - Parent template sections/fields are added as `isVisible: false` if not in current template
  - Fields from parent are merged into current sections if missing

**Key Logic**:
- Current template sections/fields: `fromTemplate: 'current'`
- Parent template sections/fields: `fromTemplate: 'parent'`, `isVisible: false` (if not in current)

---

## Database Schema

### 1. `resumeTemplates` Table
```prisma
model resumeTemplates {
  id                String                   @id @default(uuid()) @db.Uuid
  name              String
  isDefault         Boolean                  @default(false) @map("is_default")
  organizationId    String?                  @map("organization_id") @db.Uuid
  parentTemplateId  String?                  @map("parent_template_id") @db.Uuid
  organization      Organizations?           @relation(fields: [organizationId], references: [id])
  parentTemplate    resumeTemplates?         @relation("TemplateHierarchy", fields: [parentTemplateId], references: [id])
  childTemplates    resumeTemplates[]        @relation("TemplateHierarchy")
  createdAt         DateTime                 @default(now()) @map("created_at")
  updatedAt         DateTime                 @updatedAt @map("updated_at")
  deletedAt         DateTime?                @map("deleted_at")
  templateSections  resumeTemplateSections[]
  templateStructure String?                  @db.Text
  watermarkEnabled  Boolean                  @default(false) @map("watermark_enabled")
  watermarkType     watermark_type?          @map("watermark_type")

  @@map("resume_templates")
}

enum watermark_type {
  LOGO
  ORGANIZATION_NAME
}
```

**Key Fields**:
- `isDefault`: Marks system-wide default templates (organizationId must be null)
- `organizationId`: null for default templates, UUID for organization-specific templates
- `parentTemplateId`: Enables template inheritance/hierarchy
- `templateStructure`: JSON/text structure for template rendering

### 2. `resumeTemplateSections` Table
```prisma
model resumeTemplateSections {
  id             String                       @id @default(uuid()) @db.Uuid
  templateId     String                       @map("template_id") @db.Uuid
  sectionId      String                       @map("section_id") @db.Uuid
  order          Int
  isVisible      Boolean                      @default(true) @map("is_visible")
  isMandatory    Boolean                      @default(false) @map("is_mandatory")
  createdAt      DateTime                     @default(now()) @map("created_at")
  updatedAt      DateTime                     @updatedAt @map("updated_at")
  template       resumeTemplates              @relation(fields: [templateId], references: [id], onDelete: Cascade)
  section        masterResumeTemplateSections @relation(fields: [sectionId], references: [id])
  templateFields resumeTemplateFields[]

  @@unique([templateId, sectionId])
  @@map("template_sections")
}
```

**Purpose**: Maps master sections to a specific template with visibility and ordering.

### 3. `resumeTemplateFields` Table
```prisma
model resumeTemplateFields {
  id                String                     @id @default(uuid()) @db.Uuid
  templateSectionId String                     @map("template_section_id") @db.Uuid
  fieldId           String                     @map("field_id") @db.Uuid
  order             Int
  isVisible         Boolean                    @default(true) @map("is_visible")
  isMandatory       Boolean                    @default(false) @map("is_mandatory")
  customLabel       String?                    @map("custom_label")
  createdAt         DateTime                   @default(now()) @map("created_at")
  updatedAt         DateTime                   @updatedAt @map("updated_at")
  templateSection   resumeTemplateSections     @relation(fields: [templateSectionId], references: [id], onDelete: Cascade)
  field             masterResumeTemplateFields @relation(fields: [fieldId], references: [id])

  @@unique([templateSectionId, fieldId])
  @@map("template_fields")
}
```

**Purpose**: Maps master fields to template sections with visibility, ordering, and optional custom labels.

### 4. `masterResumeTemplateSections` Table (Master Data)
```prisma
model masterResumeTemplateSections {
  id               String                       @id @default(uuid()) @db.Uuid
  name             String                       @unique
  order            Int
  key              String?
  description      String?
  recordType       String                       @map("record_type")
  renderType       String                       @map("render_type")
  createdAt        DateTime                     @default(now()) @map("created_at")
  updatedAt        DateTime                     @updatedAt @map("updated_at")
  deletedAt        DateTime?                    @map("deleted_at")
  fields           masterResumeTemplateFields[]
  templateSections resumeTemplateSections[]

  @@map("master_sections")
}
```

**Purpose**: Defines available resume sections (e.g., "Personal Information", "Work Experience", "Education").

**Key Fields**:
- `recordType`: Type of records in this section (e.g., "single", "array")
- `renderType`: How to render the section (e.g., "list", "table", "paragraph")
- `key`: Unique identifier for the section

### 5. `masterResumeTemplateFields` Table (Master Data)
```prisma
model masterResumeTemplateFields {
  id             String                       @id @default(uuid()) @db.Uuid
  sectionId      String                       @map("section_id") @db.Uuid
  name           String
  fieldPath      String                       @map("field_path")
  order          Int
  dataType       String                       @map("data_type") // text, number, date, boolean, array
  createdAt      DateTime                     @default(now()) @map("created_at")
  updatedAt      DateTime                     @updatedAt @map("updated_at")
  deletedAt      DateTime?                    @map("deleted_at")
  section        masterResumeTemplateSections @relation(fields: [sectionId], references: [id])
  templateFields resumeTemplateFields[]

  @@unique([sectionId, name])
  @@map("master_fields")
}
```

**Purpose**: Defines available fields within each section.

**Key Fields**:
- `fieldPath`: Path to the field in the candidate data model (e.g., "Candidates.name", "Candidates.workExperience.companyName")
- `dataType`: Type of data (text, number, date, boolean, array)

---

## Data Flow

### Request Flow
```
Frontend Request
    ↓
GET /resume-builder/get-available-template
    ↓
resumeBuilder.controller.getAvailableTemplate()
    ↓
resumeBuilder.dao.getAvailableTemplate()
    ↓
1. Find default template (isDefault=true, organizationId=null)
    ↓
2. Call getTemplateWithInheritance(templateId)
    ↓
3. Fetch current template sections & fields
    ↓
4. Fetch parent template sections & fields (if parentTemplateId exists)
    ↓
5. Merge sections/fields with inheritance logic
    ↓
6. Return complete template structure
```

### Response Structure
The response includes:
```typescript
{
  id: string,
  name: string,
  isDefault: boolean,
  organizationId: string | null,
  parentTemplateId: string | null,
  templateStructure: string | null,  // From parent template
  watermarkEnabled: boolean,
  watermarkType: watermark_type | null,
  sections: [
    {
      id: string,                    // Master section ID
      name: string,                   // Master section name
      description: string,
      order: number,
      recordType: string,
      renderType: string,
      isVisible: boolean,
      isMandatory: boolean,
      fromTemplate: 'current' | 'parent',
      key: string,
      fields: [
        {
          id: string,                 // Master field ID
          name: string,                // Master field name
          fieldPath: string,           // e.g., "Candidates.name"
          dataType: string,
          order: number,
          isVisible: boolean,
          isMandatory: boolean,
          customLabel: string | null,
          fromTemplate: 'current' | 'parent'
        }
      ]
    }
  ]
}
```

---

## Key Features

### 1. Template Inheritance
- Templates can have a `parentTemplateId` to inherit sections/fields
- Child templates override parent settings
- Missing fields from parent are added as `isVisible: false`

### 2. Master Data System
- `masterResumeTemplateSections`: Defines available sections
- `masterResumeTemplateFields`: Defines available fields
- Templates reference master data, allowing reuse and consistency

### 3. Visibility Control
- Sections and fields can be marked as `isVisible: false`
- `isMandatory` flag controls required fields
- Custom labels can override master field names

### 4. Default Template
- System-wide default template: `isDefault=true` AND `organizationId=null`
- Used when no organization-specific template exists
- Retrieved by `get-available-template` endpoint

---

## Frontend Integration

### Service Call
```typescript
// Location: app-frontend/services/user-management-service/user-management.service.ts:3512-3525
public async fetchResumeTemplateDetails() {
    const response: IApiResponse<any> = await this.api.getRequest({
        apiPath: `/resume-builder/get-available-template`,
    });

    if (response.code !== 'S200') {
        throw new ApiException({
            message: response.message,
            code: response.code,
            data: response.data,
        });
    }
    return response.data;
}
```

---

## Related Files

1. **Routes**: `billingusers-service/src/api/resume-builder/resumeBuilder.routes.ts`
2. **Controller**: `billingusers-service/src/api/resume-builder/resumeBuilder.controller.ts`
3. **DAO**: `billingusers-service/src/api/resume-builder/resumeBuilder.dao.ts`
4. **Schema**: `billingusers-service/prisma/schema.prisma` (lines 5805-5896)
5. **Frontend Service**: `app-frontend/services/user-management-service/user-management.service.ts`

