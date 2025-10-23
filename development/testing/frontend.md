# Frontend Testing Guide

## Overview
This guide provides step-by-step instructions for testing the complete InfraR deployment workflow.

## Prerequisites

### Backend Services Running
1. **InfraR Platform API** (port 8080)
   ```bash
   cd /home/alexb/projects/infrar/infrar-platform
   go run cmd/api/main.go
   ```

2. **Cost Intelligence Service** (port 8001)
   ```bash
   cd /home/alexb/projects/infrar/infrar-cost-intelligence
   python3 -m uvicorn main:app --host 0.0.0.0 --port 8001
   ```

3. **Frontend Dev Server** (port 3000)
   ```bash
   cd /home/alexb/projects/infrar/infrar-web
   npm run dev
   ```

### Test User Account
- Email: `test@example.com`
- Password: `password123`

Or create a new account via `/register`

## Complete Deployment Workflow Test

### Step 1: Login
1. Navigate to `http://localhost:3000/login`
2. Enter credentials and login
3. **Verify**: Redirects to `/dashboard`

### Step 2: Create New Project
1. On dashboard, click "Create New Project"
2. Fill in:
   - Name: `Test Deployment`
   - Description: `Testing the full deployment workflow`
3. Click "Create Project"
4. **Verify**:
   - Redirects to project detail page `/projects/{id}`
   - Project name and description displayed
   - Status badge shows "created"

### Step 3: Upload Code
1. On project detail page, paste sample code into textarea:
   ```python
   def lambda_handler(event, context):
       return {
           'statusCode': 200,
           'body': 'Hello from InfraR!'
       }
   ```
2. Click "Upload Code" button
3. **Verify**:
   - Success message: "✓ Code uploaded and ready for deployment"
   - Green highlighted box appears in Quick Actions sidebar
   - "Start Deployment" button visible
   - **CRITICAL**: Reload the page (F5) and verify button still appears

### Step 4: Select Cloud Provider
1. Click "Start Deployment" button
2. **Verify**: Redirects to `/projects/{id}/deploy/select-provider`
3. Review provider options:
   - AWS card
   - GCP card
   - Azure card
4. **Verify**:
   - Recommended badge on cheapest option
   - Monthly cost displayed
   - Service breakdown shown
   - Progress bar shows "Step 1 of 4: Select Provider"
5. Select a provider (e.g., AWS)
6. Click "Continue" button

### Step 5: Configure Variables
1. **Verify**: Redirects to `/projects/{id}/deploy/configure-variables`
2. **Verify**: Progress bar shows "Step 2 of 4: Configure"
3. Review form fields (varies by provider):
   - **AWS**: region, runtime, memory_size, timeout
   - **GCP**: region, runtime, memory_mb
   - **Azure**: location, runtime, plan_type
4. Fill in values or use defaults
5. Click "Save & Continue"

### Step 6: Setup Credentials
1. **Verify**: Redirects to `/projects/{id}/deploy/setup-credentials`
2. **Verify**: Progress bar shows "Step 3 of 4: Credentials"
3. Review split layout:
   - **Left panel**: Step-by-step setup instructions
   - **Right panel**: Credential input form
4. Fill in credentials (provider-specific):
   - **AWS**:
     - AWS Access Key ID
     - AWS Secret Access Key
   - **GCP**:
     - Upload service account JSON file
     - Project ID
   - **Azure**:
     - Subscription ID
     - Client ID
     - Client Secret
     - Tenant ID
5. **Verify**: Security note displayed
6. Click "Continue to Deploy"

### Step 7: Provision Infrastructure
1. **Verify**: Redirects to `/projects/{id}/deploy/provision`
2. **Verify**: Progress bar shows "Step 4 of 4: Deploy"
3. Click "Deploy Infrastructure" button
4. **Verify**:
   - Provisioning status shown
   - Loading spinner appears
   - Blue progress bar animates
   - Page polls for status every 5 seconds
5. Wait for deployment to complete (2-5 minutes)
6. **Verify** on success:
   - Green success box: "Infrastructure Deployed Successfully!"
   - Created resources list displayed
   - Terraform outputs shown (if any)
   - "View Dashboard" button appears
7. **Verify** on failure:
   - Red error box with error details
   - "Retry Deployment" button appears

### Step 8: View Infrastructure Dashboard
1. Click "View Dashboard" button
2. **Verify**: Redirects to `/projects/{id}/infrastructure`
3. **Verify** deployment card shows:
   - Provider icon and name
   - Status badge (deployed)
   - Deployment ID
   - Created timestamp
   - Deployed timestamp
   - Resources list with green indicators
   - Terraform outputs
   - "Destroy" button visible
4. **Verify**: Auto-refresh indicator (refreshes every 5 seconds if deployments are active)

### Step 9: Destroy Infrastructure (Optional)
1. On infrastructure dashboard, click "Destroy" button
2. **Verify**: Confirmation dialog appears
3. Confirm destruction
4. **Verify**:
   - "Destroying..." status shown
   - Auto-refresh resumes
5. Wait for destruction to complete
6. **Verify** on success:
   - Status changes to "destroyed"
   - Destroyed timestamp shown
   - "Destroy" button no longer visible

### Step 10: Navigate Back to Project
1. Click "Back to Project" button
2. **Verify**: Returns to project detail page
3. **Verify** in Quick Actions:
   - If deployed: "View Infrastructure" button shown (green)
   - If not deployed: "Start Deployment" button shown (blue)

## Critical Bug Fix Validation

### Test: Deployment Button Persistence After Page Reload
This test validates the fix for the reported issue: "Once I upload the code, there is no way to continue with deployment process."

**Steps**:
1. Create new project
2. Upload code
3. **Verify**: "Start Deployment" button appears in green highlighted box
4. **DO NOT CLICK** the button yet
5. **Reload the page** (F5 or Ctrl+R)
6. **CRITICAL VERIFICATION**:
   - ✅ "Start Deployment" button should STILL be visible
   - ✅ Green highlighted box should still be present
   - ✅ Message "✓ Code uploaded! Ready to deploy" should be displayed
7. Click "Start Deployment" button
8. **Verify**: Deployment workflow starts successfully

**Expected Result**: Button persists after reload and deployment workflow is accessible.

**Previous Bug**: Button would disappear after reload, blocking users from continuing.

## UI Component Tests

### LoadingSpinner
- Small size used in buttons
- Medium size used in cards
- Large size used in full-page loading states

### StatusBadge
Test all status values:
- `created` → Gray
- `uploaded` → Blue
- `configured` → Cyan
- `ready` → Green
- `provisioning` → Yellow (animated)
- `deployed` → Green
- `destroying` → Orange (animated)
- `destroyed` → Red
- `failed` → Red

### ProgressBar
- Verify all 4 steps highlighted correctly
- Current step should be bold
- Completed steps should show checkmark
- Pending steps should be gray

### ProviderIcon
- AWS: Orange background
- GCP: Blue background
- Azure: Blue background

## Error Handling Tests

### Invalid Code Upload
1. Try uploading empty code
2. **Verify**: Error message: "Please enter some code"

### Invalid Credentials
1. Enter invalid cloud credentials
2. **Verify**: API error displayed in red banner

### Network Failure Simulation
1. Stop backend service
2. Try any operation
3. **Verify**: Appropriate error message shown

### Failed Deployment
1. Use invalid credentials to trigger terraform failure
2. **Verify**:
   - Status changes to "failed"
   - Error message displayed
   - "Retry Deployment" button appears

## Performance Tests

### Polling Efficiency
1. Start deployment
2. **Verify**: Polling interval is 5 seconds (not too frequent)
3. **Verify**: Polling stops when deployment reaches final state

### Auto-refresh Control
1. On infrastructure dashboard with all deployments in final state
2. **Verify**: Auto-refresh stops automatically
3. Click manual "Refresh" button
4. **Verify**: Data updates immediately

## Accessibility Tests

### Keyboard Navigation
1. Navigate through workflow using only Tab/Enter keys
2. **Verify**: All buttons and inputs are accessible

### Error Messages
1. Trigger various errors
2. **Verify**: Error messages are clear and actionable

### Loading States
1. During async operations
2. **Verify**: Loading indicators provide feedback
3. **Verify**: Buttons are disabled during operations

## Browser Compatibility
Test on:
- Chrome (latest)
- Firefox (latest)
- Safari (latest)
- Edge (latest)

## Mobile Responsiveness
Test on:
- Mobile viewport (375px width)
- Tablet viewport (768px width)
- Desktop viewport (1024px+ width)

**Verify**:
- Grid layouts adapt correctly
- Buttons remain usable
- Text remains readable

## Known Limitations
1. **Polling vs WebSockets**: Current implementation uses 5-second polling. Consider WebSockets for production for better real-time updates.
2. **Credential Storage**: Credentials are currently sent to backend. Ensure backend encrypts and securely stores them.
3. **Error Recovery**: Failed deployments require manual retry. Consider automatic retry logic.
4. **Deployment Concurrency**: Frontend doesn't prevent multiple simultaneous deployments per project.

## Test Summary Checklist

- [ ] User registration and login works
- [ ] Project creation works
- [ ] Code upload works
- [ ] **CRITICAL**: Deployment button persists after page reload
- [ ] Provider selection works with cost comparison
- [ ] Variable configuration form generates dynamically
- [ ] Credential setup shows provider-specific instructions
- [ ] Deployment provisioning works with status polling
- [ ] Infrastructure dashboard displays all deployments
- [ ] Resource and output display works
- [ ] Infrastructure destruction works
- [ ] Navigation between pages works correctly
- [ ] All status badges display correctly
- [ ] Progress bar updates through workflow
- [ ] Error messages display for failed operations
- [ ] Loading states show during async operations
- [ ] Auto-refresh works for active deployments
- [ ] Auto-refresh stops for completed deployments

## Success Criteria
✅ All checklist items pass
✅ No console errors during workflow
✅ **Deployment button bug is fixed** (persists after reload)
✅ Complete deployment workflow executes end-to-end
✅ Build completes with 0 errors

## Report Issues
If any test fails, document:
1. Steps to reproduce
2. Expected behavior
3. Actual behavior
4. Browser/environment details
5. Console errors (if any)
