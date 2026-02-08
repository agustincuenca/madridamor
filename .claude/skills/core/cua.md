# Skill: Computer Use Agent (CUA)

## Purpose
Test applications by navigating them as a real user would, using visual interaction with the browser.

## Overview

The Computer Use Agent (CUA) allows automated testing by:
- Taking screenshots of the application
- Clicking on elements visually
- Typing text into inputs
- Navigating through flows
- Validating visual output

## Testing Workflow

### 1. Start the Application
```bash
# Ensure the app is running
rails server -p 3000
```

### 2. Navigate and Test
```markdown
## Test: User Registration Flow

1. Open browser to http://localhost:3000
2. Click "Sign Up" link
3. Fill in the registration form:
   - Email: test@example.com
   - Password: password123
   - Confirm Password: password123
4. Click "Create Account" button
5. Verify success message appears
6. Verify redirect to dashboard
```

### 3. Visual Verification
```markdown
## Verify: Article List Page

1. Navigate to /articles
2. Screenshot shows:
   - Navigation bar at top
   - List of article cards
   - Each card has title, excerpt, author
   - "New Article" button visible (if logged in)
3. Click on first article
4. Verify article detail page loads
```

## Test Scenarios

### Authentication Flow
```markdown
### Login Test
1. Go to /session/new
2. Enter email: user@example.com
3. Enter password: password123
4. Click "Sign In"
5. Verify: Dashboard page loads
6. Verify: User name appears in navigation

### Logout Test
1. Click user menu in navigation
2. Click "Sign Out"
3. Verify: Redirected to home page
4. Verify: "Sign In" link appears
```

### CRUD Operations
```markdown
### Create Article
1. Navigate to /articles/new
2. Fill in:
   - Title: "Test Article"
   - Body: "This is test content"
3. Click "Create Article"
4. Verify: Success message
5. Verify: Article page shows entered content

### Edit Article
1. On article page, click "Edit"
2. Change title to "Updated Title"
3. Click "Update Article"
4. Verify: Title updated on page

### Delete Article
1. On article page, click "Delete"
2. Confirm deletion dialog
3. Verify: Redirected to articles list
4. Verify: Article no longer in list
```

### Form Validation
```markdown
### Required Field Validation
1. Go to /articles/new
2. Leave title empty
3. Click "Create Article"
4. Verify: Error message "Title can't be blank"
5. Verify: Form still shows entered content

### Email Format Validation
1. Go to /registration/new
2. Enter invalid email: "notanemail"
3. Submit form
4. Verify: Error message about email format
```

### Navigation Testing
```markdown
### Mobile Navigation
1. Resize window to 375px width
2. Verify: Hamburger menu icon visible
3. Click hamburger menu
4. Verify: Navigation menu slides in
5. Click "Articles"
6. Verify: Menu closes, navigates to articles

### Breadcrumb Navigation
1. Go to /articles/1
2. Verify: Breadcrumb shows "Home > Articles > Article Title"
3. Click "Articles" in breadcrumb
4. Verify: Navigates to articles list
```

### Interactive Features
```markdown
### Turbo Frame Update
1. Go to /articles
2. Click "Edit" on inline-editable item
3. Verify: Form appears without page reload
4. Update content
5. Click "Save"
6. Verify: Content updates without page reload

### Real-time Updates
1. Open /articles in two browser tabs
2. In Tab 1, create new article
3. Verify: Article appears in Tab 2 automatically
```

### Accessibility Testing
```markdown
### Keyboard Navigation
1. Press Tab to move through interactive elements
2. Verify: Focus indicators visible
3. Press Enter on focused button
4. Verify: Action executes

### Screen Reader Check
1. Enable screen reader
2. Navigate through page
3. Verify: All content announced correctly
4. Verify: Form labels read properly
```

## Reporting Format

### Test Report Structure
```markdown
# Test Report: [Feature Name]
Date: YYYY-MM-DD
Tester: CUA

## Summary
- Total Tests: X
- Passed: X
- Failed: X

## Test Results

### ✅ Test 1: [Name]
- Steps completed successfully
- Expected behavior confirmed

### ❌ Test 2: [Name]
- **Failed at step**: [step number]
- **Expected**: [what should happen]
- **Actual**: [what happened]
- **Screenshot**: [description of visual state]

## Issues Found
1. [Issue description]
   - Severity: High/Medium/Low
   - Steps to reproduce
   - Suggested fix

## Recommendations
- [Recommendation 1]
- [Recommendation 2]
```

## Common Test Patterns

### Login Before Tests
```markdown
## Setup: Authenticate User
1. Go to /session/new
2. Login as test user
3. Verify logged in state
4. Continue with tests...
```

### Data Setup
```markdown
## Setup: Create Test Data
1. Login as admin
2. Create test article via UI
3. Note the article ID
4. Proceed with tests using this data
```

### Cleanup
```markdown
## Cleanup: Remove Test Data
1. Navigate to created resources
2. Delete each test resource
3. Verify deletion
4. Logout
```

## Visual Checks

### Layout Verification
```markdown
## Check: Page Layout
- [ ] Header present and styled correctly
- [ ] Main content centered
- [ ] Footer at bottom
- [ ] No horizontal scroll
- [ ] Responsive at mobile widths
```

### Style Verification
```markdown
## Check: Component Styling
- [ ] Buttons have correct colors
- [ ] Links are distinguishable
- [ ] Forms are aligned
- [ ] Error states are red
- [ ] Success states are green
```

### Content Verification
```markdown
## Check: Content Display
- [ ] Headings hierarchy correct
- [ ] Text readable (size, contrast)
- [ ] Images load and display
- [ ] No placeholder text visible
- [ ] Dates formatted correctly
```

## Error Handling Tests

### Network Errors
```markdown
### Offline Behavior
1. Go to /articles
2. Disable network
3. Try to create article
4. Verify: Appropriate error message shown
5. Re-enable network
6. Verify: App recovers gracefully
```

### Server Errors
```markdown
### 500 Error Page
1. Trigger server error (if testable)
2. Verify: Custom error page shows
3. Verify: User-friendly message
4. Verify: Navigation still works
```

## Best Practices

1. **Start fresh** - Clear session before each test suite
2. **Take screenshots** - Document visual state at key points
3. **Test mobile** - Always verify responsive behavior
4. **Check accessibility** - Use keyboard, verify focus
5. **Verify data** - Confirm changes persist correctly
6. **Test edge cases** - Empty states, long content, special characters
