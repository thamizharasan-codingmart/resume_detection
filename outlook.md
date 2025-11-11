# Outlook Mail Integration Implementation Plan

This document outlines the step-by-step implementation plan for integrating Outlook mail functionality, mirroring all Gmail integration features.

## Overview

The Outlook integration will use **Microsoft Graph API** (similar to how Gmail uses Google APIs) to provide the same functionality as the existing Gmail integration.

---

## Stage 1: Authentication & OAuth Setup

### 1.1 Microsoft Azure App Registration
- [ ] Register application in Azure Portal
- [ ] Configure redirect URIs
- [ ] Set required API permissions:
  - `Mail.ReadWrite` - Read and write mail
  - `Mail.Send` - Send mail
  - `Mail.Read` - Read mail
  - `offline_access` - Refresh tokens
- [ ] Generate Client ID and Client Secret
- [ ] Store credentials in config (similar to `googleClientId`, `googleClientSecret`)

### 1.2 OAuth Flow Implementation
**File**: `src/utils/microsoft.ts` (new file, similar to `google.ts`)

**Methods to implement**:
- [ ] `exchangeCode(code: string)` - Exchange authorization code for tokens
- [ ] `getUserProfile(accessToken: string)` - Get user email/profile
- [ ] `checkTokenValidity(accessToken: string)` - Validate access token
- [ ] `refreshAccessToken(refreshToken: string)` - Refresh expired tokens
- [ ] `getTokenScopes(accessToken: string)` - Get token scopes

**Constants**:
- [ ] Add `outlookScopes` in `src/utils/constants.ts`:
  ```typescript
  export const outlookScopes = [
    'https://graph.microsoft.com/Mail.ReadWrite',
    'https://graph.microsoft.com/Mail.Send',
    'offline_access'
  ];
  ```

### 1.3 Mailbox Connection
**File**: `src/api/mail/mail.controller.ts`

**Endpoint**: `POST /link-mailbox`
- [ ] Add provider check (GMAIL vs OUTLOOK)
- [ ] Implement Outlook OAuth flow (similar to `linkMailBox`)
- [ ] Store credentials in `mailBoxCredentials` table
- [ ] Set `provider: 'OUTLOOK'` in `mailBox` table

**File**: `src/api/mail/mail.dao.ts`
- [ ] Update `storeCredentials` to support OUTLOOK provider
- [ ] Update `getTokenDetails` to handle Outlook token refresh
- [ ] Ensure provider-specific logic where needed

---

## Stage 2: Mail Reading & Listing

### 2.1 List All Mails
**File**: `src/utils/microsoft.ts`

**Method**: `listAllMails(token, filters)`
- [ ] Use Microsoft Graph API: `GET /me/messages`
- [ ] Support filters:
  - Label/Folder (Inbox, Sent, Drafts, etc.)
  - Date range
  - Search query
  - Pagination (top, skip)
- [ ] Map Outlook folder structure to Gmail labels
- [ ] Return formatted response similar to Gmail format

**File**: `src/api/mail/mail.controller.ts`
- [ ] Update `listAllMails` to check provider and route accordingly
- [ ] Add Outlook-specific handling

### 2.2 Get Mail Content
**Method**: `getMessageContent(token, messageId)`
- [ ] Use: `GET /me/messages/{messageId}`
- [ ] Parse MIME content
- [ ] Extract body (HTML/text)
- [ ] Extract attachments metadata
- [ ] Return formatted response

**Method**: `getMailContent(token, messageId)`
- [ ] Similar to above but with full details
- [ ] Include thread/conversation context

### 2.3 Get Thread Messages
**Method**: `getThreadAllMessages(token, threadId)`
- [ ] Use: `GET /me/messages/{messageId}/conversation`
- [ ] Or use conversationId to get all messages in thread
- [ ] Return all messages in conversation

**File**: `src/api/mail/mail.controller.ts`
- [ ] Update `getThreadAllMessages` for Outlook

### 2.4 Get Emails by Thread IDs
**Method**: `getEmailsByThreadIds(token, threadIds, label, from?, to?)`
- [ ] Fetch multiple conversations
- [ ] Apply label/folder filtering
- [ ] Filter by from/to if provided
- [ ] Return formatted list

---

## Stage 3: Mail Sending

### 3.1 Send Mail
**Method**: `sendEmail(token, emailData)`
- [ ] Use: `POST /me/sendMail`
- [ ] Support:
  - To, CC, BCC recipients
  - Subject and body (HTML/text)
  - Attachments
  - Draft creation
- [ ] Handle MIME encoding for attachments
- [ ] Return messageId and threadId

**File**: `src/api/mail/mail.controller.ts`
- [ ] Update `sendMail` to route based on provider

### 3.2 Reply to Message
**Method**: `replyToEmail(token, messageId, isReplyAll, options)`
- [ ] Use: `POST /me/messages/{messageId}/reply` or `/replyAll`
- [ ] Include original message in reply
- [ ] Support attachments
- [ ] Handle CC/BCC for reply-all
- [ ] Return sent message details

**File**: `src/api/mail/mail.controller.ts`
- [ ] Update `replyToMessage` for Outlook

### 3.3 Forward Mail
**Method**: `forwardMail(token, messageId, forwardData)`
- [ ] Use: `POST /me/messages/{messageId}/forward`
- [ ] Include original message content
- [ ] Support attachments
- [ ] Support CC/BCC
- [ ] Return forwarded message details

**File**: `src/api/mail/mail.controller.ts`
- [ ] Update `forwardMail` for Outlook

### 3.4 Draft Management
**Method**: `createDraft(token, draftData)`
- [ ] Use: `POST /me/messages` with `isDraft: true`
- [ ] Or use: `POST /me/drafts`
- [ ] Support all email fields
- [ ] Return draftId

**Method**: `updateDraft(token, draftId, draftData)`
- [ ] Use: `PATCH /me/drafts/{draftId}`
- [ ] Update draft content

**Method**: `deleteDraft(token, draftId)`
- [ ] Use: `DELETE /me/drafts/{draftId}`
- [ ] Return success status

**File**: `src/api/mail/mail.controller.ts`
- [ ] Update `addToDraft` and `deleteDraft` for Outlook

---

## Stage 4: Labels & Folders

### 4.1 List All Labels/Folders
**Method**: `listAllLabels(token)`
- [ ] Use: `GET /me/mailFolders`
- [ ] Map Outlook folders to Gmail label structure
- [ ] Include system folders (Inbox, Sent, Drafts, etc.)
- [ ] Include custom folders
- [ ] Return formatted list

**File**: `src/api/mail/mail.controller.ts`
- [ ] Update `listAllLabels` for Outlook

### 4.2 Create Label/Folder
**Method**: `createLabel(token, labelData)`
- [ ] Use: `POST /me/mailFolders`
- [ ] Support nested folders (parentFolderId)
- [ ] Return created folder details

**File**: `src/api/mail/mail.controller.ts`
- [ ] Update `createLabel` for Outlook

### 4.3 Remove Label/Folder
**Method**: `deleteLabel(token, folderId)`
- [ ] Use: `DELETE /me/mailFolders/{folderId}`
- [ ] Handle errors (cannot delete system folders)
- [ ] Return success status

**File**: `src/api/mail/mail.controller.ts`
- [ ] Update `removeLabel` for Outlook

### 4.4 Modify Mail Labels (Move to Folder)
**Method**: `modifyMailLabel(token, messageId, folderId, action)`
- [ ] Use: `POST /me/messages/{messageId}/move`
- [ ] Support moving messages between folders
- [ ] Handle add/remove label actions (Outlook uses folders)
- [ ] Return updated message

**File**: `src/api/mail/mail.controller.ts`
- [ ] Update `modifyMailLabel` for Outlook

---

## Stage 5: Mail Management

### 5.1 Move Message to Trash
**Method**: `moveMessageToTrash(token, messageId)`
- [ ] Use: `POST /me/messages/{messageId}/move` with DeletedItems folder
- [ ] Or use: `DELETE /me/messages/{messageId}` (soft delete)
- [ ] Return success status

**File**: `src/api/mail/mail.controller.ts`
- [ ] Update `moveMessageToTrash` for Outlook

### 5.2 Mark as Read/Unread
**Method**: `markAsRead(token, messageId, isRead)`
- [ ] Use: `PATCH /me/messages/{messageId}`
- [ ] Update `isRead` property
- [ ] Return updated message

### 5.3 Flag/Unflag Messages
**Method**: `flagMessage(token, messageId, flagData)`
- [ ] Use: `PATCH /me/messages/{messageId}`
- [ ] Update `flag` property
- [ ] Support flag colors and due dates

---

## Stage 6: Real-time Notifications (Webhooks)

### 6.1 Subscription Setup
**Method**: `createOutlookSubscription(token, notificationUrl)`
- [ ] Use: `POST /subscriptions`
- [ ] Configure webhook endpoint
- [ ] Set expiration (max 3 days, need renewal)
- [ ] Store subscription details in database
- [ ] Return subscriptionId and expiration

**File**: `src/services/outlook-watch.service.ts` (new file)
- [ ] Similar to `gmail-watch.service.ts`
- [ ] Implement subscription creation
- [ ] Implement subscription renewal
- [ ] Handle subscription expiration

### 6.2 Webhook Handler
**File**: `src/api/mail/mail.controller.ts`

**Endpoint**: `POST /outlook-webhook`
- [ ] Verify webhook signature (if applicable)
- [ ] Parse notification payload
- [ ] Extract messageId and changeType
- [ ] Queue processing job
- [ ] Return 202 Accepted

**File**: `src/services/outlook-webhook-worker.service.ts` (new file)
- [ ] Process webhook notifications
- [ ] Fetch changed messages
- [ ] Handle resume detection (if auto-import enabled)
- [ ] Update mailbox sync status

### 6.3 Subscription Renewal
**File**: `src/services/outlook-watch.service.ts`
- [ ] Implement cron job to renew expiring subscriptions
- [ ] Check subscriptions expiring within 24 hours
- [ ] Renew each subscription
- [ ] Update database with new expiration

---

## Stage 7: Attachments Handling

### 7.1 Download Attachments
**Method**: `downloadAttachment(token, messageId, attachmentId)`
- [ ] Use: `GET /me/messages/{messageId}/attachments/{attachmentId}`
- [ ] Decode base64 content
- [ ] Return attachment buffer and metadata

### 7.2 Upload Attachments
- [ ] Handle multipart/form-data in send/reply endpoints
- [ ] Convert to base64 for Graph API
- [ ] Include in message payload

**File**: `src/utils/microsoft.ts`
- [ ] Implement attachment encoding/decoding utilities

---

## Stage 8: Token Management & Refresh

### 8.1 Token Refresh Logic
**File**: `src/api/mail/mail.dao.ts`
- [ ] Update `getTokenDetails` to handle Outlook tokens
- [ ] Check token expiration
- [ ] Refresh if expired using Microsoft refresh token endpoint
- [ ] Update database with new tokens

**File**: `src/utils/microsoft.ts`
- [ ] Implement `refreshAccessToken` using Microsoft OAuth2 endpoint
- [ ] Handle refresh token expiration
- [ ] Return new access token and expiry

### 8.2 Token Validation
**Method**: `checkTokenValidity(accessToken)`
- [ ] Use: `GET /me` or token introspection endpoint
- [ ] Verify token is valid and not expired
- [ ] Return validity status

---

## Stage 9: Database Schema Updates

### 9.1 Schema Verification
**File**: `prisma/schema.prisma`
- [x] Verify `mail_provider` enum includes `OUTLOOK` (already exists)
- [ ] Ensure `mailBox` table supports OUTLOOK provider
- [ ] Ensure `mailBoxCredentials` stores Outlook tokens correctly

### 9.2 Migration (if needed)
- [ ] Create migration if schema changes required
- [ ] Test migration on development database

---

## Stage 10: Service Layer Abstraction

### 10.1 Mail Service Interface
**File**: `src/services/mail-provider.service.ts` (new file)
- [ ] Create interface/abstract class for mail providers
- [ ] Define common methods:
  - `sendEmail()`
  - `listMails()`
  - `getMessage()`
  - `createLabel()`
  - `refreshToken()`
  - etc.
- [ ] Implement Gmail provider (wrap existing google.ts)
- [ ] Implement Outlook provider (wrap microsoft.ts)

### 10.2 Provider Factory
**File**: `src/services/mail-provider.factory.ts` (new file)
- [ ] Factory to get correct provider based on `mail_provider` enum
- [ ] Return appropriate service instance

**File**: `src/api/mail/mail.controller.ts`
- [ ] Update all methods to use factory
- [ ] Remove provider-specific conditionals

---

## Stage 11: Error Handling & Logging

### 11.1 Error Handling
- [ ] Map Microsoft Graph API errors to application errors
- [ ] Handle rate limiting (429 errors)
- [ ] Handle token expiration gracefully
- [ ] Handle network errors

### 11.2 Logging
- [ ] Add logging for all Outlook API calls
- [ ] Log token refresh operations
- [ ] Log webhook processing
- [ ] Use existing logger utility

---

## Stage 12: Testing

### 12.1 Unit Tests
- [ ] Test OAuth flow
- [ ] Test token refresh
- [ ] Test mail sending/receiving
- [ ] Test label/folder operations
- [ ] Test webhook handling

### 12.2 Integration Tests
- [ ] Test end-to-end mailbox connection
- [ ] Test mail sync
- [ ] Test webhook notifications
- [ ] Test error scenarios

### 12.3 Manual Testing
- [ ] Test with real Outlook account
- [ ] Test all endpoints
- [ ] Test edge cases
- [ ] Test performance

---

## Stage 13: Configuration & Environment

### 13.1 Environment Variables
**File**: `src/config/config.ts`
- [ ] Add `microsoftClientId`
- [ ] Add `microsoftClientSecret`
- [ ] Add `microsoftRedirectUri`
- [ ] Add `outlookWebhookSecret` (if applicable)
- [ ] Add `outlookSubscriptionRenewalInterval`

### 13.2 Constants
**File**: `src/utils/constants.ts`
- [ ] Add `outlookScopes`
- [ ] Add Outlook API endpoints
- [ ] Add folder mappings (Outlook → Gmail labels)

---

## Stage 14: Documentation & Deployment

### 14.1 API Documentation
- [ ] Update Swagger/OpenAPI docs
- [ ] Document Outlook-specific endpoints
- [ ] Document differences from Gmail

### 14.2 Deployment Checklist
- [ ] Verify Azure app registration
- [ ] Verify redirect URIs in production
- [ ] Set up webhook endpoint URL
- [ ] Configure subscription renewal cron
- [ ] Monitor logs and errors

---

## Implementation Priority

### Phase 1: Core Functionality (Weeks 1-2)
1. Stage 1: Authentication & OAuth Setup
2. Stage 2: Mail Reading & Listing
3. Stage 3: Mail Sending (basic)

### Phase 2: Advanced Features (Weeks 3-4)
4. Stage 4: Labels & Folders
5. Stage 5: Mail Management
6. Stage 8: Token Management

### Phase 3: Real-time & Integration (Weeks 5-6)
7. Stage 6: Real-time Notifications
8. Stage 7: Attachments Handling
9. Stage 10: Service Layer Abstraction

### Phase 4: Polish & Deploy (Week 7)
10. Stage 11: Error Handling & Logging
11. Stage 12: Testing
12. Stage 13: Configuration
13. Stage 14: Documentation

---

## Key Differences: Gmail vs Outlook

| Feature | Gmail API | Microsoft Graph API |
|---------|-----------|---------------------|
| **Authentication** | Google OAuth2 | Microsoft OAuth2 |
| **Labels** | Labels system | Folders system |
| **Threads** | ThreadId | ConversationId |
| **Watch** | Pub/Sub webhooks | Webhook subscriptions (3-day expiry) |
| **Token Refresh** | Google endpoint | Microsoft endpoint |
| **Base URL** | `https://gmail.googleapis.com` | `https://graph.microsoft.com/v1.0` |

---

## Microsoft Graph API Endpoints Reference

### Authentication
- Token: `https://login.microsoftonline.com/common/oauth2/v2.0/token`
- Authorization: `https://login.microsoftonline.com/common/oauth2/v2.0/authorize`

### Mail Operations
- List messages: `GET /me/messages`
- Get message: `GET /me/messages/{id}`
- Send mail: `POST /me/sendMail`
- Reply: `POST /me/messages/{id}/reply`
- Forward: `POST /me/messages/{id}/forward`
- Move: `POST /me/messages/{id}/move`
- Delete: `DELETE /me/messages/{id}`

### Folders
- List folders: `GET /me/mailFolders`
- Create folder: `POST /me/mailFolders`
- Delete folder: `DELETE /me/mailFolders/{id}`

### Subscriptions
- Create: `POST /subscriptions`
- Renew: `PATCH /subscriptions/{id}`
- Delete: `DELETE /subscriptions/{id}`

---

## Notes

1. **Webhook Expiration**: Outlook subscriptions expire after 3 days maximum. Implement automatic renewal.

2. **Rate Limiting**: Microsoft Graph API has rate limits. Implement retry logic with exponential backoff.

3. **Folder vs Labels**: Outlook uses folders instead of labels. Map Outlook folders to Gmail label concept for consistency.

4. **Conversation Threading**: Outlook uses `conversationId` instead of `threadId`. Map appropriately.

5. **Attachment Size**: Check Microsoft Graph API limits for attachment sizes.

6. **Batch Operations**: Consider using batch requests for multiple operations to reduce API calls.

---

## Resources

- [Microsoft Graph API Documentation](https://docs.microsoft.com/en-us/graph/api/overview)
- [Microsoft Graph Mail API](https://docs.microsoft.com/en-us/graph/api/resources/mail-api-overview)
- [Microsoft Authentication](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-auth-code-flow)
- [Microsoft Graph Webhooks](https://docs.microsoft.com/en-us/graph/api/resources/webhooks)

---

**Last Updated**: [Current Date]
**Status**: Planning Phase
**Assigned To**: [Team/Developer]

