# McKinney-Vento LMS Reports - Task List with Micro-Steps

## Current Issues Summary
Based on the quick-guide.txt and code analysis:
1. Tables are divs styled to look like tables (not responsive)
2. Navigation shows incorrect data when drilling down
3. No breadcrumb navigation for non-admin users
4. State management issues on refresh
5. UI/UX inconsistencies and popup styling issues
6. Report menu item not highlighted correctly

## Task List with Micro-Steps

### Task 1: Replace ReportTable with David's Table Component
**Priority:** High
**Estimated Time:** 4-6 hours

#### Micro-steps:
1. **Analyze current ReportTable usage** (30 min)
   - [ ] Map all props used in ReportTable.vue
   - [ ] Document all events emitted
   - [ ] List all data transformations needed

2. **Create adapter for Table component** (1 hour)
   - [ ] Map ReportTable props to Table component props
   - [ ] Transform data structure to match Table component format
   - [ ] Handle special cases (color coding, percentage displays)

3. **Replace in LocationReports.vue** (2 hours)
   - [ ] Import David's Table component
   - [ ] Convert column definitions to Table format
   - [ ] Set up API mode configuration
   - [ ] Implement search functionality
   - [ ] Add filters for report types

4. **Add custom components for special columns** (1 hour)
   - [ ] Create PercentageDisplay component for completion percentages
   - [ ] Create ColorIndicator component for threshold warnings
   - [ ] Integrate with Table's custom column type

5. **Test and refine** (30 min)
   - [ ] Verify data displays correctly
   - [ ] Check responsive behavior
   - [ ] Ensure pagination works

### Task 2: Fix State Coordinator Navigation Logic
**Priority:** High
**Estimated Time:** 3-4 hours

#### Micro-steps:
1. **Map current navigation flow** (30 min)
   - [ ] Document state → region → district → school flow
   - [ ] Identify where wrong IDs are passed
   - [ ] Check URL parameter handling

2. **Fix drill-down navigation** (1.5 hours)
   - [ ] Fix "View Region" button to pass correct region ID
   - [ ] Fix "View Liaisons" to show region-specific liaisons
   - [ ] Fix "View Staff" to show region-specific staff
   - [ ] Ensure district and school levels work correctly

3. **Implement proper URL parameter handling** (1 hour)
   - [ ] Use route params correctly for each level
   - [ ] Validate IDs before making API calls
   - [ ] Handle invalid IDs gracefully

4. **Add row actions using Table component** (1 hour)
   - [ ] Configure urlResolver for each action
   - [ ] Use conditional display based on user permissions
   - [ ] Test navigation between levels

### Task 3: Comprehensive UX/Navigation Improvements
**Priority:** High
**Estimated Time:** 4-5 hours
**Based on David's feedback: Focus on general UX/navigation, page headers, breadcrumbs, and layout optimization**

#### Micro-steps:
1. **Add missing page headers and context indicators** (1.5 hours)
   - [ ] Create PageHeader component showing current location (e.g., "Region: North Central > Essential Staff")
   - [ ] Add clear section headers to ES/Liaison report pages
   - [ ] Display current user's context and access level
   - [ ] Show data scope (what region/district/school is being viewed)

2. **Implement breadcrumb navigation** (1 hour)
   - [ ] Create BreadcrumbNav component with proper hierarchy
   - [ ] Add to all report views (state/region/district/school)
   - [ ] Handle dynamic path building and navigation clicks
   - [ ] Style according to design system

3. **Redesign page layout and information hierarchy** (1.5 hours)
   - [ ] Rethink divs/headings/action buttons placement
   - [ ] Improve visual hierarchy with better spacing and typography
   - [ ] Optimize action button placement and grouping
   - [ ] Ensure responsive design works across devices

4. **Enhance navigation UX** (1 hour)
   - [ ] Add "Back to [Parent Level]" buttons where appropriate
   - [ ] Improve visual feedback for current location
   - [ ] Add contextual help text explaining current view
   - [ ] Test navigation flow across all user types

### Task 4: Add Bulk Actions for Reports
**Priority:** Low (Deferred)
**Estimated Time:** 2-3 hours
**Note:** Deferred based on David's feedback to focus on UX/navigation first

#### Micro-steps:
1. **Define bulk action requirements** (30 min)
   - [ ] Identify which reports need bulk actions
   - [ ] Define actions (export selected, email reminders, etc.)

2. **Implement bulk selection** (1 hour)
   - [ ] Enable allowBulkActions prop
   - [ ] Handle selection events
   - [ ] Show selection count

3. **Add toolbar actions** (1 hour)
   - [ ] Use toolbarLeft slot for bulk actions
   - [ ] Add export selected functionality
   - [ ] Add batch email reminders (if needed)

4. **Test bulk operations** (30 min)
   - [ ] Verify selection across pages
   - [ ] Test action execution
   - [ ] Handle errors gracefully

### Task 5: Fix Report Menu Highlighting
**Priority:** Low (Deferred)
**Estimated Time:** 1 hour
**Note:** Deferred to focus on critical UX improvements first

#### Micro-steps:
1. **Identify navigation component** (15 min)
   - [ ] Find where menu highlighting is handled
   - [ ] Check route matching logic

2. **Fix active state detection** (30 min)
   - [ ] Update route matching for /reports paths
   - [ ] Remove incorrect Dashboard highlighting

3. **Test across all report views** (15 min)
   - [ ] Verify highlighting works for all report types
   - [ ] Check sub-navigation highlighting

### Task 6: Responsive Design Improvements
**Priority:** Medium
**Estimated Time:** 2 hours

#### Micro-steps:
1. **Test current responsive behavior** (30 min)
   - [ ] Check mobile view
   - [ ] Test tablet view
   - [ ] Document specific issues

2. **Apply Table component responsive features** (1 hour)
   - [ ] Configure sticky columns for mobile
   - [ ] Set appropriate column widths
   - [ ] Test horizontal scrolling

3. **Optimize mobile UI** (30 min)
   - [ ] Adjust button sizes for mobile
   - [ ] Improve filter dropdown on mobile
   - [ ] Test touch interactions

### Task 7: Handle State Management on Refresh
**Priority:** Medium
**Estimated Time:** 2 hours

#### Micro-steps:
1. **Identify state loss issues** (30 min)
   - [ ] Document which data is lost on refresh
   - [ ] Check current state management approach

2. **Implement persistent state** (1 hour)
   - [ ] Use URL params for filter state
   - [ ] Store necessary data in sessionStorage
   - [ ] Restore state on component mount

3. **Test refresh scenarios** (30 min)
   - [ ] Test filter persistence
   - [ ] Test navigation state
   - [ ] Verify data reloads correctly

## Development Approach

### Phase 1 (Week 1):
- Task 1: Replace ReportTable with Table component
- Task 2: Fix State Coordinator navigation

### Phase 2 (Week 1-2):
- Task 3: Implement breadcrumb navigation
- Task 5: Fix menu highlighting

### Phase 3 (Week 2):
- Task 4: Add bulk actions
- Task 6: Responsive improvements
- Task 7: State management fixes

## Testing Strategy
1. **Unit tests** for new components
2. **Integration tests** for navigation flow
3. **Manual testing** with different user roles:
   - Admin
   - State Coordinator
   - Regional Coordinator
   - District Liaison
   - School Liaison

## Success Criteria
- ✅ All tables use David's Table component
- ✅ Navigation shows correct data at each level
- ✅ Breadcrumb navigation works for all users
- ✅ Reports are fully responsive
- ✅ State persists on refresh
- ✅ Menu highlighting is correct
- ✅ Bulk actions work where needed