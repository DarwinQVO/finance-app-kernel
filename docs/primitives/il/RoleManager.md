# RoleManager (IL Component)

**Layer:** Interaction Layer (IL)
**Domain:** Security & Access (Vertical 5.4)
**Status:** Draft
**Last Updated:** 2025-10-25

---

## Table of Contents

1. [Overview](#overview)
2. [Component Interface](#component-interface)
3. [Visual Design](#visual-design)
4. [State Management](#state-management)
5. [Data Fetching](#data-fetching)
6. [Rendering Logic](#rendering-logic)
7. [Event Handlers](#event-handlers)
8. [Accessibility](#accessibility)
9. [Performance](#performance)
10. [Multi-Domain Examples](#multi-domain-examples)
11. [Integration](#integration)
12. [Testing Strategy](#testing-strategy)

---

## Overview

### Purpose

The **RoleManager** is a comprehensive role-based access control (RBAC) interface component that enables security administrators to assign roles, manage permissions, and visualize role hierarchies across tenant organizations. It provides an intuitive UI for creating custom roles, previewing permission impacts, and auditing role changes. This component serves as the central hub for access control governance, ensuring principle of least privilege and separation of duties.

### Key Capabilities

- **Visual Role Hierarchy**: Tree view showing role inheritance and nesting
- **User Role Assignment**: Drag-and-drop or dialog-based role assignment to users
- **Permission Preview**: Real-time preview of effective permissions for selected role
- **Custom Role Creation**: Build new roles by combining existing permissions
- **Bulk Operations**: Assign/revoke roles for multiple users simultaneously
- **Role Templates**: Pre-defined role templates (Admin, Editor, Viewer, etc.)
- **Audit Trail**: Track all role changes with user, timestamp, and reason
- **Conflict Detection**: Identify permission conflicts (e.g., Read + Deny on same resource)
- **Time-Based Roles**: Assign temporary roles with expiration dates
- **Multi-Tenant Support**: Manage roles across multiple tenant organizations
- **Search & Filter**: Find users by email, role, department, or permission
- **Export Capabilities**: Export role assignments for compliance reporting

### Problem Statement

Access control management is complex and error-prone:

1. **Role Sprawl**: Organizations accumulate 100+ roles over time, creating confusion
2. **Over-Privileged Users**: Users granted Admin role "temporarily" for years
3. **Permission Drift**: User permissions accumulate without regular review
4. **Compliance Gaps**: Can't prove who had access to sensitive data at specific times
5. **Manual Processes**: Excel spreadsheets for role assignments lead to errors
6. **No Visibility**: Can't see effective permissions for a user across all roles
7. **Slow Onboarding**: Takes days to provision correct roles for new hires
8. **Shadow Admin**: Users gain Admin through undocumented role combinations

### Solution

The RoleManager provides:

- **Visual Clarity**: See entire role hierarchy at a glance
- **Impact Preview**: Preview permission changes before applying
- **Audit Trail**: Immutable log of every role assignment/revocation
- **Automated Workflows**: Onboarding templates grant correct roles automatically
- **Permission Calculator**: Compute effective permissions across multiple roles
- **Expiration Enforcement**: Auto-revoke temporary roles on expiration
- **Conflict Alerts**: Warn when assigning conflicting permissions
- **Compliance Reports**: One-click export for SOC 2, ISO 27001, etc.

This transforms access control from reactive cleanup to proactive governance.

---

## Component Interface

### Primary Props

```typescript
interface RoleManagerProps {
  // ============================================================
  // Tenant & User Context
  // ============================================================

  /**
   * Tenant ID(s) to manage roles for
   * Multiple tenants for cross-tenant admin views
   */
  tenantId: string | string[]

  /**
   * Current user (for permission checks)
   */
  currentUser: {
    id: string
    email: string
    roles: string[]
    permissions: string[]
  }

  /**
   * Enable editing (vs. read-only view)
   * @default true
   */
  editable?: boolean

  // ============================================================
  // Display Configuration
  // ============================================================

  /**
   * Layout mode
   * @default 'split' (hierarchy + users)
   */
  layout?: 'split' | 'hierarchy-only' | 'users-only' | 'compact'

  /**
   * Show permission preview panel
   * @default true
   */
  showPermissionPreview?: boolean

  /**
   * Show audit trail panel
   * @default true
   */
  showAuditTrail?: boolean

  /**
   * Enable role hierarchy tree view
   * @default true
   */
  enableHierarchyView?: boolean

  /**
   * Enable bulk operations
   * @default true
   */
  enableBulkOperations?: boolean

  /**
   * Enable custom role creation
   * @default true (if user has permission)
   */
  enableCustomRoles?: boolean

  /**
   * Enable time-based (temporary) roles
   * @default true
   */
  enableTemporaryRoles?: boolean

  /**
   * Theme
   * @default 'auto'
   */
  theme?: 'light' | 'dark' | 'auto'

  // ============================================================
  // Role Configuration
  // ============================================================

  /**
   * Available roles to assign
   * If not provided, fetched from API
   */
  availableRoles?: Role[]

  /**
   * Role templates for quick assignment
   */
  roleTemplates?: RoleTemplate[]

  /**
   * Hide specific roles from UI
   */
  hiddenRoles?: string[]

  /**
   * Disabled roles (show but can't assign)
   */
  disabledRoles?: string[]

  /**
   * Role hierarchy override
   * Defines parent-child relationships
   */
  roleHierarchy?: Record<string, string[]> // parentRole -> childRoles

  // ============================================================
  // Permission Configuration
  // ============================================================

  /**
   * Available permissions
   * If not provided, fetched from API
   */
  availablePermissions?: Permission[]

  /**
   * Group permissions by category
   */
  permissionCategories?: PermissionCategory[]

  /**
   * Show permission descriptions
   * @default true
   */
  showPermissionDescriptions?: boolean

  /**
   * Warn on dangerous permissions
   * @default true
   */
  warnDangerousPermissions?: boolean

  /**
   * Dangerous permission identifiers
   */
  dangerousPermissions?: string[] // e.g., ['delete_all', 'grant_admin']

  // ============================================================
  // Filtering & Search
  // ============================================================

  /**
   * Initial search query
   */
  initialSearchQuery?: string

  /**
   * Initial filters
   */
  initialFilters?: {
    roles?: string[]
    departments?: string[]
    status?: ('active' | 'inactive')[]
  }

  /**
   * Enable advanced search
   * @default true
   */
  enableAdvancedSearch?: boolean

  // ============================================================
  // Callbacks
  // ============================================================

  /**
   * Called when role is assigned to user
   */
  onRoleAssign?: (params: {
    userId: string
    roleId: string
    expiresAt?: Date
    reason?: string
  }) => void | Promise<void>

  /**
   * Called when role is revoked from user
   */
  onRoleRevoke?: (params: {
    userId: string
    roleId: string
    reason?: string
  }) => void | Promise<void>

  /**
   * Called when custom role is created
   */
  onRoleCreate?: (role: Role) => void | Promise<void>

  /**
   * Called when role is updated
   */
  onRoleUpdate?: (roleId: string, updates: Partial<Role>) => void | Promise<void>

  /**
   * Called when role is deleted
   */
  onRoleDelete?: (roleId: string) => void | Promise<void>

  /**
   * Called when user is selected
   */
  onUserSelect?: (user: User) => void

  /**
   * Called when role is selected in hierarchy
   */
  onRoleSelect?: (role: Role) => void

  /**
   * Called when permission conflict detected
   */
  onConflictDetected?: (conflict: PermissionConflict) => void

  /**
   * Called when export is initiated
   */
  onExport?: (format: 'csv' | 'pdf' | 'json', data: any) => void

  // ============================================================
  // Validation
  // ============================================================

  /**
   * Custom validation before role assignment
   * Return error message if invalid, null if valid
   */
  validateRoleAssignment?: (params: {
    user: User
    role: Role
  }) => string | null | Promise<string | null>

  /**
   * Require reason for role changes
   * @default false
   */
  requireChangeReason?: boolean

  /**
   * Require approval for sensitive roles
   * @default false
   */
  requireApproval?: boolean

  /**
   * Sensitive roles that require approval
   */
  sensitiveRoles?: string[] // e.g., ['Admin', 'Security_Manager']

  // ============================================================
  // Integration
  // ============================================================

  /**
   * API client for role operations
   */
  apiClient?: RoleAPIClient

  /**
   * Enable integration with AuditViewer
   * @default true
   */
  enableAuditViewerLink?: boolean

  /**
   * Enable integration with SecurityDashboard
   * @default true
   */
  enableSecurityDashboardLink?: boolean

  /**
   * Custom actions to show in toolbar
   */
  customActions?: React.ReactNode

  // ============================================================
  // Performance
  // ============================================================

  /**
   * Enable virtualization for large user lists
   * @default true
   */
  enableVirtualization?: boolean

  /**
   * Page size for user list
   * @default 50
   */
  userPageSize?: number

  /**
   * Debounce search input (ms)
   * @default 300
   */
  searchDebounceMs?: number
}
```

### Supporting Types

```typescript
/**
 * Role definition
 */
interface Role {
  id: string
  name: string
  displayName: string
  description: string
  permissions: string[] // Permission IDs
  inheritsFrom?: string[] // Parent role IDs
  color?: string // For visual distinction
  icon?: string // Icon name
  system: boolean // System role (can't delete)
  tenantId: string
  createdAt: Date
  updatedAt: Date
  createdBy: string
  metadata?: Record<string, any>
}

/**
 * Permission definition
 */
interface Permission {
  id: string
  name: string
  displayName: string
  description: string
  category: string
  dangerous: boolean // Requires extra confirmation
  resourceType?: string
  actions?: ('read' | 'write' | 'delete' | 'admin')[]
}

/**
 * Permission category for grouping
 */
interface PermissionCategory {
  id: string
  name: string
  description: string
  permissions: string[] // Permission IDs
}

/**
 * User with role assignments
 */
interface User {
  id: string
  email: string
  name: string
  avatar?: string
  department?: string
  status: 'active' | 'inactive'
  roles: UserRole[]
  effectivePermissions?: string[] // Computed from roles
  tenantId: string
  createdAt: Date
  lastLoginAt?: Date
}

/**
 * User role assignment
 */
interface UserRole {
  roleId: string
  roleName: string
  assignedAt: Date
  assignedBy: string
  expiresAt?: Date
  reason?: string
  status: 'active' | 'expired' | 'pending_approval'
}

/**
 * Role template for quick assignment
 */
interface RoleTemplate {
  id: string
  name: string
  description: string
  roles: string[] // Role IDs to assign
  icon?: string
  category: string // e.g., 'Engineering', 'Sales', 'Support'
}

/**
 * Permission conflict
 */
interface PermissionConflict {
  userId: string
  permission: string
  conflictType: 'allow_deny' | 'duplicate' | 'over_privileged'
  roles: string[] // Conflicting roles
  message: string
  severity: 'warning' | 'error'
}

/**
 * Role change audit entry
 */
interface RoleAuditEntry {
  id: string
  timestamp: Date
  userId: string
  userEmail: string
  action: 'assign' | 'revoke' | 'create' | 'update' | 'delete'
  roleId: string
  roleName: string
  targetUserId?: string // User affected by change
  targetUserEmail?: string
  reason?: string
  metadata?: Record<string, any>
}

/**
 * API client interface
 */
interface RoleAPIClient {
  getRoles(tenantId: string): Promise<Role[]>
  getUsers(params: {
    tenantId: string
    search?: string
    filters?: any
    page?: number
    pageSize?: number
  }): Promise<{ users: User[]; total: number }>
  getPermissions(tenantId: string): Promise<Permission[]>
  assignRole(params: {
    userId: string
    roleId: string
    expiresAt?: Date
    reason?: string
  }): Promise<void>
  revokeRole(params: { userId: string; roleId: string; reason?: string }): Promise<void>
  createRole(role: Omit<Role, 'id' | 'createdAt' | 'updatedAt'>): Promise<Role>
  updateRole(roleId: string, updates: Partial<Role>): Promise<Role>
  deleteRole(roleId: string): Promise<void>
  getAuditLog(params: { tenantId: string; limit?: number }): Promise<RoleAuditEntry[]>
  exportData(params: { format: 'csv' | 'pdf' | 'json'; data: any }): Promise<Blob>
}
```

---

## Visual Design

### ASCII Wireframe (Split Layout)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ Role Manager                                          🏢 Tenant: Acme Corp   │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  🔍 Search: [________________]  Filter: [Roles ▼] [Dept ▼]  📤 Export ▼     │
│                                                                              │
├─────────────────────────────────┬────────────────────────────────────────────┤
│                                 │                                            │
│ 🌳 Role Hierarchy               │ 👥 Users (234)                             │
│                                 │                                            │
│  ┌─ Admin                       │  ┌──────────────────────────────────────┐ │
│  │  ├─ Security Manager         │  │ ☑ Name           Roles      Status  │ │
│  │  └─ Tenant Admin             │  ├──────────────────────────────────────┤ │
│  ├─ Editor                      │  │ ☐ Alice Johnson  Admin       Active  │ │
│  │  ├─ Content Editor           │  │ ☐ Bob Smith      Editor      Active  │ │
│  │  └─ Report Editor            │  │ ☐ Carol White    Viewer      Active  │ │
│  └─ Viewer                      │  │ ☐ Dave Brown     Editor      Inactive│ │
│     ├─ Read-Only                │  │ ☐ Eve Davis      Viewer      Active  │ │
│     └─ Guest                    │  └──────────────────────────────────────┘ │
│                                 │                                            │
│  [+ Create Role]                │  < 1 2 3 ... 10 >                          │
│                                 │                                            │
│                                 │  Selected: Alice Johnson                   │
│                                 │  ┌──────────────────────────────────────┐ │
│                                 │  │ Current Roles:                       │ │
│                                 │  │  • Admin (expires: Never)            │ │
│                                 │  │  • Security_Manager (expires: Never) │ │
│                                 │  │                                      │ │
│                                 │  │ [Assign Role ▼]  [Revoke Selected]  │ │
│                                 │  └──────────────────────────────────────┘ │
│                                 │                                            │
├─────────────────────────────────┴────────────────────────────────────────────┤
│                                                                              │
│ 📋 Permission Preview: Admin                                                 │
│                                                                              │
│  Documents (4)      Reports (3)       Users (5)        System (2)           │
│  ─────────────      ────────────      ──────────       ──────────           │
│  ✓ Read             ✓ Read            ✓ Read           ✓ Read               │
│  ✓ Write            ✓ Write           ✓ Write          ✓ Write              │
│  ✓ Delete           ✓ Delete          ✓ Delete         ⚠️ Delete All       │
│  ✓ Share            ✓ Publish         ⚠️ Grant Admin   ⚠️ Config           │
│                                                                              │
│  ⚠️ 2 dangerous permissions detected                                         │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Compact Layout (Mobile)

```
┌─────────────────────────────┐
│ Role Manager          ≡     │
├─────────────────────────────┤
│                             │
│ 🔍 [________________]       │
│                             │
│ 👥 Users (234)              │
│ ┌─────────────────────────┐ │
│ │ Alice Johnson           │ │
│ │ Admin, Security_Manager │ │
│ │ [View] [Edit]           │ │
│ ├─────────────────────────┤ │
│ │ Bob Smith               │ │
│ │ Editor                  │ │
│ │ [View] [Edit]           │ │
│ └─────────────────────────┘ │
│                             │
│ [Show Hierarchy]            │
│                             │
└─────────────────────────────┘
```

---

## State Management

### React State Hooks

```typescript
import { useState, useEffect, useCallback, useMemo } from 'react'
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'

export function useRoleManager(props: RoleManagerProps) {
  const {
    tenantId,
    currentUser,
    editable = true,
    layout = 'split',
    showPermissionPreview = true,
    showAuditTrail = true,
    enableHierarchyView = true,
    enableBulkOperations = true,
    enableCustomRoles = true,
    enableTemporaryRoles = true,
    roleHierarchy,
    initialSearchQuery = '',
    initialFilters,
    requireChangeReason = false,
    requireApproval = false,
    sensitiveRoles = [],
    onRoleAssign,
    onRoleRevoke,
    onRoleCreate,
    onRoleUpdate,
    onRoleDelete,
    onUserSelect,
    onRoleSelect,
    onConflictDetected,
    validateRoleAssignment,
    apiClient,
    userPageSize = 50,
    searchDebounceMs = 300,
  } = props

  // ============================================================
  // Local State
  // ============================================================

  const [searchQuery, setSearchQuery] = useState(initialSearchQuery)
  const [filters, setFilters] = useState(initialFilters || {})
  const [selectedUsers, setSelectedUsers] = useState<string[]>([])
  const [selectedRole, setSelectedRole] = useState<Role | null>(null)
  const [selectedUser, setSelectedUser] = useState<User | null>(null)
  const [currentPage, setCurrentPage] = useState(1)
  const [assignDialogOpen, setAssignDialogOpen] = useState(false)
  const [createRoleDialogOpen, setCreateRoleDialogOpen] = useState(false)
  const [expandedRoles, setExpandedRoles] = useState<Set<string>>(new Set())

  // ============================================================
  // Memoized Values
  // ============================================================

  const tenantIdArray = useMemo(() => {
    return Array.isArray(tenantId) ? tenantId : [tenantId]
  }, [tenantId])

  const canEdit = useMemo(() => {
    return editable && currentUser.permissions.includes('manage_roles')
  }, [editable, currentUser.permissions])

  const canCreateRoles = useMemo(() => {
    return (
      enableCustomRoles &&
      canEdit &&
      currentUser.permissions.includes('create_roles')
    )
  }, [enableCustomRoles, canEdit, currentUser.permissions])

  // ============================================================
  // Callbacks
  // ============================================================

  const handleUserSelect = useCallback(
    (user: User) => {
      setSelectedUser(user)
      onUserSelect?.(user)
    },
    [onUserSelect]
  )

  const handleRoleSelect = useCallback(
    (role: Role) => {
      setSelectedRole(role)
      onRoleSelect?.(role)
    },
    [onRoleSelect]
  )

  const handleBulkSelect = useCallback((userIds: string[]) => {
    setSelectedUsers(userIds)
  }, [])

  const handleSearchChange = useCallback(
    (query: string) => {
      setSearchQuery(query)
      setCurrentPage(1) // Reset to first page
    },
    []
  )

  const handleFilterChange = useCallback((newFilters: any) => {
    setFilters(newFilters)
    setCurrentPage(1)
  }, [])

  const toggleRoleExpansion = useCallback((roleId: string) => {
    setExpandedRoles((prev) => {
      const next = new Set(prev)
      if (next.has(roleId)) {
        next.delete(roleId)
      } else {
        next.add(roleId)
      }
      return next
    })
  }, [])

  return {
    // State
    searchQuery,
    filters,
    selectedUsers,
    selectedRole,
    selectedUser,
    currentPage,
    assignDialogOpen,
    createRoleDialogOpen,
    expandedRoles,
    tenantIdArray,
    canEdit,
    canCreateRoles,

    // Actions
    setSearchQuery: handleSearchChange,
    setFilters: handleFilterChange,
    setSelectedUsers: handleBulkSelect,
    setSelectedRole: handleRoleSelect,
    setSelectedUser: handleUserSelect,
    setCurrentPage,
    setAssignDialogOpen,
    setCreateRoleDialogOpen,
    toggleRoleExpansion,
  }
}
```

---

## Data Fetching

### React Query Integration

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { RoleAPIClient } from './api/RoleAPIClient'

/**
 * Fetch roles for tenant
 */
export function useRoles(tenantId: string, enabled = true) {
  return useQuery({
    queryKey: ['roles', tenantId],
    queryFn: async () => {
      const client = new RoleAPIClient()
      return client.getRoles(tenantId)
    },
    enabled,
    staleTime: 5 * 60 * 1000, // 5 minutes
  })
}

/**
 * Fetch users with pagination and filtering
 */
export function useUsers(params: {
  tenantId: string
  search?: string
  filters?: any
  page?: number
  pageSize?: number
}) {
  return useQuery({
    queryKey: ['users', params],
    queryFn: async () => {
      const client = new RoleAPIClient()
      return client.getUsers({
        tenantId: params.tenantId,
        search: params.search,
        filters: params.filters,
        page: params.page || 1,
        pageSize: params.pageSize || 50,
      })
    },
    keepPreviousData: true, // Keep old data while fetching new page
    staleTime: 30 * 1000, // 30 seconds
  })
}

/**
 * Fetch available permissions
 */
export function usePermissions(tenantId: string, enabled = true) {
  return useQuery({
    queryKey: ['permissions', tenantId],
    queryFn: async () => {
      const client = new RoleAPIClient()
      return client.getPermissions(tenantId)
    },
    enabled,
    staleTime: 10 * 60 * 1000, // 10 minutes
  })
}

/**
 * Mutation: Assign role to user
 */
export function useAssignRole(onSuccess?: () => void) {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: async (params: {
      userId: string
      roleId: string
      expiresAt?: Date
      reason?: string
    }) => {
      const client = new RoleAPIClient()
      return client.assignRole(params)
    },
    onSuccess: () => {
      // Invalidate users query to refetch
      queryClient.invalidateQueries({ queryKey: ['users'] })
      onSuccess?.()
    },
  })
}

/**
 * Mutation: Revoke role from user
 */
export function useRevokeRole(onSuccess?: () => void) {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: async (params: {
      userId: string
      roleId: string
      reason?: string
    }) => {
      const client = new RoleAPIClient()
      return client.revokeRole(params)
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] })
      onSuccess?.()
    },
  })
}

/**
 * Mutation: Create custom role
 */
export function useCreateRole(onSuccess?: (role: Role) => void) {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: async (role: Omit<Role, 'id' | 'createdAt' | 'updatedAt'>) => {
      const client = new RoleAPIClient()
      return client.createRole(role)
    },
    onSuccess: (role) => {
      queryClient.invalidateQueries({ queryKey: ['roles'] })
      onSuccess?.(role)
    },
  })
}

/**
 * Mutation: Update role
 */
export function useUpdateRole(onSuccess?: () => void) {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: async ({
      roleId,
      updates,
    }: {
      roleId: string
      updates: Partial<Role>
    }) => {
      const client = new RoleAPIClient()
      return client.updateRole(roleId, updates)
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['roles'] })
      onSuccess?.()
    },
  })
}

/**
 * Mutation: Delete role
 */
export function useDeleteRole(onSuccess?: () => void) {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: async (roleId: string) => {
      const client = new RoleAPIClient()
      return client.deleteRole(roleId)
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['roles'] })
      onSuccess?.()
    },
  })
}

/**
 * Fetch audit log
 */
export function useRoleAuditLog(params: { tenantId: string; limit?: number }) {
  return useQuery({
    queryKey: ['role-audit', params.tenantId, params.limit],
    queryFn: async () => {
      const client = new RoleAPIClient()
      return client.getAuditLog(params)
    },
    staleTime: 30 * 1000,
  })
}
```

---

## Rendering Logic

### Core JSX Structure

```tsx
import React from 'react'
import { Card, CardHeader, CardTitle, CardContent } from '@/components/ui/card'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Checkbox } from '@/components/ui/checkbox'
import { Badge } from '@/components/ui/badge'
import { Dialog, DialogContent, DialogHeader, DialogTitle } from '@/components/ui/dialog'
import { Select, SelectTrigger, SelectValue, SelectContent, SelectItem } from '@/components/ui/select'
import { Users, Shield, ChevronRight, ChevronDown, Plus, Trash2 } from 'lucide-react'

export function RoleManager(props: RoleManagerProps) {
  const state = useRoleManager(props)
  const {
    searchQuery,
    filters,
    selectedUsers,
    selectedRole,
    selectedUser,
    currentPage,
    assignDialogOpen,
    createRoleDialogOpen,
    expandedRoles,
    tenantIdArray,
    canEdit,
    canCreateRoles,
    setSearchQuery,
    setFilters,
    setSelectedUsers,
    setSelectedRole,
    setSelectedUser,
    setCurrentPage,
    setAssignDialogOpen,
    setCreateRoleDialogOpen,
    toggleRoleExpansion,
  } = state

  // Fetch data
  const rolesQuery = useRoles(tenantIdArray[0])
  const usersQuery = useUsers({
    tenantId: tenantIdArray[0],
    search: searchQuery,
    filters,
    page: currentPage,
    pageSize: props.userPageSize || 50,
  })
  const permissionsQuery = usePermissions(tenantIdArray[0])

  const roles = rolesQuery.data || []
  const users = usersQuery.data?.users || []
  const totalUsers = usersQuery.data?.total || 0
  const permissions = permissionsQuery.data || []

  const isLoading = rolesQuery.isLoading || usersQuery.isLoading

  // Build role hierarchy
  const roleHierarchy = buildRoleHierarchy(roles, props.roleHierarchy)

  return (
    <div className="role-manager w-full p-6">
      {/* Header */}
      <div className="flex items-center justify-between mb-6">
        <div className="flex items-center gap-3">
          <Shield className="h-8 w-8 text-purple-600" />
          <div>
            <h1 className="text-2xl font-bold">Role Manager</h1>
            <p className="text-sm text-gray-500">
              Manage user roles and permissions
            </p>
          </div>
        </div>

        <div className="flex items-center gap-3">
          {props.customActions}
          {canCreateRoles && (
            <Button onClick={() => setCreateRoleDialogOpen(true)}>
              <Plus className="h-4 w-4 mr-2" />
              Create Role
            </Button>
          )}
        </div>
      </div>

      {/* Search & Filters */}
      <div className="flex items-center gap-4 mb-6 p-4 bg-gray-50 rounded-lg">
        <Input
          placeholder="Search users..."
          value={searchQuery}
          onChange={(e) => setSearchQuery(e.target.value)}
          className="max-w-xs"
        />

        <Select
          value={filters.roles?.[0] || 'all'}
          onValueChange={(value) => {
            setFilters({
              ...filters,
              roles: value === 'all' ? undefined : [value],
            })
          }}
        >
          <SelectTrigger className="w-40">
            <SelectValue placeholder="Filter by role" />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value="all">All Roles</SelectItem>
            {roles.map((role) => (
              <SelectItem key={role.id} value={role.id}>
                {role.displayName}
              </SelectItem>
            ))}
          </SelectContent>
        </Select>

        {props.enableExport && (
          <Button variant="outline">Export</Button>
        )}
      </div>

      {isLoading && (
        <div className="flex items-center justify-center h-64">
          <span className="text-gray-500">Loading...</span>
        </div>
      )}

      {!isLoading && (
        <div className={`grid gap-6 ${props.layout === 'split' ? 'grid-cols-2' : 'grid-cols-1'}`}>
          {/* Role Hierarchy Panel */}
          {props.enableHierarchyView && props.layout !== 'users-only' && (
            <Card>
              <CardHeader>
                <CardTitle className="flex items-center gap-2">
                  <Shield className="h-5 w-5" />
                  Role Hierarchy
                </CardTitle>
              </CardHeader>
              <CardContent>
                <RoleHierarchyTree
                  hierarchy={roleHierarchy}
                  expandedRoles={expandedRoles}
                  selectedRole={selectedRole}
                  onRoleSelect={setSelectedRole}
                  onToggleExpand={toggleRoleExpansion}
                />
              </CardContent>
            </Card>
          )}

          {/* Users Panel */}
          {props.layout !== 'hierarchy-only' && (
            <Card>
              <CardHeader>
                <CardTitle className="flex items-center gap-2">
                  <Users className="h-5 w-5" />
                  Users ({totalUsers})
                </CardTitle>
              </CardHeader>
              <CardContent>
                <UserList
                  users={users}
                  selectedUsers={selectedUsers}
                  selectedUser={selectedUser}
                  onUserSelect={setSelectedUser}
                  onBulkSelect={setSelectedUsers}
                  enableBulkOperations={props.enableBulkOperations}
                />

                {/* Pagination */}
                <div className="flex items-center justify-between mt-4">
                  <Button
                    variant="outline"
                    disabled={currentPage === 1}
                    onClick={() => setCurrentPage((p) => p - 1)}
                  >
                    Previous
                  </Button>
                  <span className="text-sm text-gray-600">
                    Page {currentPage} of {Math.ceil(totalUsers / (props.userPageSize || 50))}
                  </span>
                  <Button
                    variant="outline"
                    disabled={currentPage * (props.userPageSize || 50) >= totalUsers}
                    onClick={() => setCurrentPage((p) => p + 1)}
                  >
                    Next
                  </Button>
                </div>

                {/* Selected User Details */}
                {selectedUser && (
                  <div className="mt-4 p-4 border rounded-lg">
                    <h4 className="font-medium mb-2">
                      Selected: {selectedUser.name}
                    </h4>
                    <div className="mb-3">
                      <span className="text-sm font-medium">Current Roles:</span>
                      <ul className="mt-1 space-y-1">
                        {selectedUser.roles.map((userRole) => (
                          <li key={userRole.roleId} className="flex items-center justify-between text-sm">
                            <span>• {userRole.roleName}</span>
                            {userRole.expiresAt && (
                              <Badge variant="outline">
                                Expires: {new Date(userRole.expiresAt).toLocaleDateString()}
                              </Badge>
                            )}
                          </li>
                        ))}
                      </ul>
                    </div>

                    {canEdit && (
                      <div className="flex gap-2">
                        <Button
                          size="sm"
                          onClick={() => setAssignDialogOpen(true)}
                        >
                          Assign Role
                        </Button>
                        <Button
                          size="sm"
                          variant="destructive"
                          disabled={selectedUser.roles.length === 0}
                        >
                          Revoke Selected
                        </Button>
                      </div>
                    )}
                  </div>
                )}
              </CardContent>
            </Card>
          )}
        </div>
      )}

      {/* Permission Preview Panel */}
      {props.showPermissionPreview && selectedRole && (
        <Card className="mt-6">
          <CardHeader>
            <CardTitle>Permission Preview: {selectedRole.displayName}</CardTitle>
          </CardHeader>
          <CardContent>
            <PermissionPreview
              role={selectedRole}
              permissions={permissions}
              showDescriptions={props.showPermissionDescriptions}
              dangerousPermissions={props.dangerousPermissions}
            />
          </CardContent>
        </Card>
      )}

      {/* Assign Role Dialog */}
      <AssignRoleDialog
        open={assignDialogOpen}
        onOpenChange={setAssignDialogOpen}
        user={selectedUser}
        availableRoles={roles}
        enableTemporaryRoles={props.enableTemporaryRoles}
        requireChangeReason={props.requireChangeReason}
        onAssign={props.onRoleAssign}
      />

      {/* Create Role Dialog */}
      <CreateRoleDialog
        open={createRoleDialogOpen}
        onOpenChange={setCreateRoleDialogOpen}
        permissions={permissions}
        onRoleCreate={props.onRoleCreate}
      />
    </div>
  )
}
```

### Sub-Components

```tsx
/**
 * Role Hierarchy Tree
 */
function RoleHierarchyTree(props: {
  hierarchy: RoleHierarchyNode[]
  expandedRoles: Set<string>
  selectedRole: Role | null
  onRoleSelect: (role: Role) => void
  onToggleExpand: (roleId: string) => void
}) {
  const { hierarchy, expandedRoles, selectedRole, onRoleSelect, onToggleExpand } = props

  return (
    <div className="space-y-1">
      {hierarchy.map((node) => (
        <RoleTreeNode
          key={node.role.id}
          node={node}
          level={0}
          expanded={expandedRoles.has(node.role.id)}
          selected={selectedRole?.id === node.role.id}
          onSelect={() => onRoleSelect(node.role)}
          onToggle={() => onToggleExpand(node.role.id)}
        />
      ))}
    </div>
  )
}

function RoleTreeNode(props: {
  node: RoleHierarchyNode
  level: number
  expanded: boolean
  selected: boolean
  onSelect: () => void
  onToggle: () => void
}) {
  const { node, level, expanded, selected, onSelect, onToggle } = props
  const hasChildren = node.children.length > 0

  return (
    <div>
      <div
        className={`flex items-center gap-2 px-3 py-2 rounded cursor-pointer hover:bg-gray-100 ${
          selected ? 'bg-blue-50 border-l-4 border-blue-600' : ''
        }`}
        style={{ paddingLeft: `${level * 20 + 12}px` }}
        onClick={onSelect}
      >
        {hasChildren && (
          <button onClick={(e) => { e.stopPropagation(); onToggle() }}>
            {expanded ? (
              <ChevronDown className="h-4 w-4" />
            ) : (
              <ChevronRight className="h-4 w-4" />
            )}
          </button>
        )}
        {!hasChildren && <span className="w-4" />}

        <Shield className="h-4 w-4 text-gray-400" />
        <span className="font-medium">{node.role.displayName}</span>
        <Badge variant="outline" className="ml-auto text-xs">
          {node.role.permissions.length}
        </Badge>
      </div>

      {expanded && hasChildren && (
        <div>
          {node.children.map((child) => (
            <RoleTreeNode
              key={child.role.id}
              node={child}
              level={level + 1}
              expanded={props.expandedRoles?.has(child.role.id) || false}
              selected={props.selectedRole?.id === child.role.id}
              onSelect={() => props.onRoleSelect(child.role)}
              onToggle={() => props.onToggleExpand(child.role.id)}
            />
          ))}
        </div>
      )}
    </div>
  )
}

/**
 * User List
 */
function UserList(props: {
  users: User[]
  selectedUsers: string[]
  selectedUser: User | null
  onUserSelect: (user: User) => void
  onBulkSelect: (userIds: string[]) => void
  enableBulkOperations?: boolean
}) {
  const { users, selectedUsers, selectedUser, onUserSelect, onBulkSelect, enableBulkOperations } = props

  return (
    <div className="border rounded-lg overflow-hidden">
      <table className="w-full text-sm">
        <thead className="bg-gray-50">
          <tr>
            {enableBulkOperations && (
              <th className="px-3 py-2 text-left w-10">
                <Checkbox
                  checked={selectedUsers.length === users.length}
                  onCheckedChange={(checked) => {
                    onBulkSelect(checked ? users.map((u) => u.id) : [])
                  }}
                />
              </th>
            )}
            <th className="px-3 py-2 text-left">Name</th>
            <th className="px-3 py-2 text-left">Roles</th>
            <th className="px-3 py-2 text-left">Status</th>
          </tr>
        </thead>
        <tbody>
          {users.map((user) => (
            <tr
              key={user.id}
              className={`border-t hover:bg-gray-50 cursor-pointer ${
                selectedUser?.id === user.id ? 'bg-blue-50' : ''
              }`}
              onClick={() => onUserSelect(user)}
            >
              {enableBulkOperations && (
                <td className="px-3 py-2">
                  <Checkbox
                    checked={selectedUsers.includes(user.id)}
                    onCheckedChange={(checked) => {
                      const next = checked
                        ? [...selectedUsers, user.id]
                        : selectedUsers.filter((id) => id !== user.id)
                      onBulkSelect(next)
                    }}
                    onClick={(e) => e.stopPropagation()}
                  />
                </td>
              )}
              <td className="px-3 py-2">
                <div>
                  <div className="font-medium">{user.name}</div>
                  <div className="text-xs text-gray-500">{user.email}</div>
                </div>
              </td>
              <td className="px-3 py-2">
                <div className="flex flex-wrap gap-1">
                  {user.roles.slice(0, 2).map((role) => (
                    <Badge key={role.roleId} variant="outline">
                      {role.roleName}
                    </Badge>
                  ))}
                  {user.roles.length > 2 && (
                    <Badge variant="outline">+{user.roles.length - 2}</Badge>
                  )}
                </div>
              </td>
              <td className="px-3 py-2">
                <Badge variant={user.status === 'active' ? 'default' : 'secondary'}>
                  {user.status}
                </Badge>
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  )
}

/**
 * Permission Preview
 */
function PermissionPreview(props: {
  role: Role
  permissions: Permission[]
  showDescriptions?: boolean
  dangerousPermissions?: string[]
}) {
  const { role, permissions, showDescriptions, dangerousPermissions = [] } = props

  // Group permissions by category
  const permissionMap = new Map(permissions.map((p) => [p.id, p]))
  const rolePermissions = role.permissions.map((id) => permissionMap.get(id)).filter(Boolean) as Permission[]

  const categorized = rolePermissions.reduce((acc, perm) => {
    const cat = perm.category
    if (!acc[cat]) acc[cat] = []
    acc[cat].push(perm)
    return acc
  }, {} as Record<string, Permission[]>)

  const dangerousCount = rolePermissions.filter((p) => dangerousPermissions.includes(p.id)).length

  return (
    <div>
      <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
        {Object.entries(categorized).map(([category, perms]) => (
          <div key={category}>
            <h4 className="font-medium mb-2">
              {category} ({perms.length})
            </h4>
            <ul className="space-y-1 text-sm">
              {perms.map((perm) => (
                <li key={perm.id} className="flex items-center gap-1">
                  {dangerousPermissions.includes(perm.id) ? (
                    <span className="text-red-600">⚠️</span>
                  ) : (
                    <span className="text-green-600">✓</span>
                  )}
                  <span>{perm.displayName}</span>
                </li>
              ))}
            </ul>
          </div>
        ))}
      </div>

      {dangerousCount > 0 && (
        <div className="mt-4 p-3 bg-amber-50 border border-amber-200 rounded">
          <p className="text-sm text-amber-800">
            ⚠️ {dangerousCount} dangerous permission{dangerousCount > 1 ? 's' : ''} detected
          </p>
        </div>
      )}
    </div>
  )
}
```

---

## Event Handlers

### Role Assignment

```typescript
/**
 * Handle role assignment with validation
 */
async function handleRoleAssign(params: {
  userId: string
  roleId: string
  expiresAt?: Date
  reason?: string
}) {
  const { userId, roleId, expiresAt, reason } = params

  // 1. Validate if custom validation provided
  if (props.validateRoleAssignment) {
    const user = users.find((u) => u.id === userId)
    const role = roles.find((r) => r.id === roleId)

    if (user && role) {
      const error = await props.validateRoleAssignment({ user, role })
      if (error) {
        showError(error)
        return
      }
    }
  }

  // 2. Check for sensitive roles requiring approval
  if (props.requireApproval && props.sensitiveRoles?.includes(roleId)) {
    const approved = await requestApproval({
      userId,
      roleId,
      requestedBy: props.currentUser.id,
    })

    if (!approved) {
      showWarning('Role assignment requires approval')
      return
    }
  }

  // 3. Check for permission conflicts
  const user = users.find((u) => u.id === userId)
  if (user) {
    const conflicts = detectPermissionConflicts(user, roleId, roles)
    if (conflicts.length > 0) {
      conflicts.forEach((conflict) => props.onConflictDetected?.(conflict))

      const proceed = await confirmDialog({
        title: 'Permission Conflicts Detected',
        message: `Found ${conflicts.length} conflict(s). Proceed anyway?`,
      })

      if (!proceed) return
    }
  }

  // 4. Execute assignment
  const assignMutation = useAssignRole()
  await assignMutation.mutateAsync({
    userId,
    roleId,
    expiresAt,
    reason,
  })

  // 5. Trigger callback
  props.onRoleAssign?.({ userId, roleId, expiresAt, reason })

  // 6. Show success message
  showSuccess(`Role assigned successfully${expiresAt ? ' (temporary)' : ''}`)
}

/**
 * Detect permission conflicts
 */
function detectPermissionConflicts(
  user: User,
  newRoleId: string,
  allRoles: Role[]
): PermissionConflict[] {
  const conflicts: PermissionConflict[] = []

  const newRole = allRoles.find((r) => r.id === newRoleId)
  if (!newRole) return conflicts

  const userRoles = user.roles.map((ur) => allRoles.find((r) => r.id === ur.roleId)).filter(Boolean) as Role[]

  // Check for permission overlaps
  const existingPermissions = new Set(
    userRoles.flatMap((r) => r.permissions)
  )

  newRole.permissions.forEach((perm) => {
    if (existingPermissions.has(perm)) {
      conflicts.push({
        userId: user.id,
        permission: perm,
        conflictType: 'duplicate',
        roles: [newRoleId, ...userRoles.map((r) => r.id)],
        message: `Permission "${perm}" already granted by another role`,
        severity: 'warning',
      })
    }
  })

  // Check for over-privileged (e.g., Admin + another role is redundant)
  if (userRoles.some((r) => r.name === 'Admin') && newRole.name !== 'Admin') {
    conflicts.push({
      userId: user.id,
      permission: 'all',
      conflictType: 'over_privileged',
      roles: ['Admin', newRoleId],
      message: 'User already has Admin role (grants all permissions)',
      severity: 'warning',
    })
  }

  return conflicts
}
```

### Bulk Operations

```typescript
/**
 * Bulk assign roles to multiple users
 */
async function handleBulkRoleAssign(params: {
  userIds: string[]
  roleId: string
  expiresAt?: Date
  reason?: string
}) {
  const { userIds, roleId, expiresAt, reason } = params

  // Show confirmation
  const confirmed = await confirmDialog({
    title: 'Bulk Role Assignment',
    message: `Assign role to ${userIds.length} user(s)?`,
  })

  if (!confirmed) return

  // Execute assignments in parallel
  const assignMutation = useAssignRole()

  const results = await Promise.allSettled(
    userIds.map((userId) =>
      assignMutation.mutateAsync({ userId, roleId, expiresAt, reason })
    )
  )

  // Count successes/failures
  const successes = results.filter((r) => r.status === 'fulfilled').length
  const failures = results.filter((r) => r.status === 'rejected').length

  if (failures > 0) {
    showWarning(`${successes} succeeded, ${failures} failed`)
  } else {
    showSuccess(`Role assigned to ${successes} user(s)`)
  }
}

/**
 * Bulk revoke roles
 */
async function handleBulkRoleRevoke(params: {
  userIds: string[]
  roleId: string
  reason?: string
}) {
  const { userIds, roleId, reason } = params

  const confirmed = await confirmDialog({
    title: 'Bulk Role Revocation',
    message: `Revoke role from ${userIds.length} user(s)?`,
    variant: 'destructive',
  })

  if (!confirmed) return

  const revokeMutation = useRevokeRole()

  const results = await Promise.allSettled(
    userIds.map((userId) =>
      revokeMutation.mutateAsync({ userId, roleId, reason })
    )
  )

  const successes = results.filter((r) => r.status === 'fulfilled').length
  const failures = results.filter((r) => r.status === 'rejected').length

  if (failures > 0) {
    showWarning(`${successes} succeeded, ${failures} failed`)
  } else {
    showSuccess(`Role revoked from ${successes} user(s)`)
  }
}
```

---

## Accessibility

### WCAG 2.1 AA Compliance

```typescript
/**
 * Accessibility features:
 * 1. Keyboard navigation for role tree
 * 2. ARIA labels for interactive elements
 * 3. Focus management for dialogs
 * 4. Screen reader announcements
 */

// Keyboard navigation for role tree
function useRoleTreeKeyboard(
  roles: Role[],
  selectedRole: Role | null,
  onRoleSelect: (role: Role) => void
) {
  useEffect(() => {
    function handleKeyDown(e: KeyboardEvent) {
      if (!selectedRole) return

      const currentIndex = roles.findIndex((r) => r.id === selectedRole.id)

      if (e.key === 'ArrowDown') {
        e.preventDefault()
        const nextIndex = (currentIndex + 1) % roles.length
        onRoleSelect(roles[nextIndex])
      } else if (e.key === 'ArrowUp') {
        e.preventDefault()
        const prevIndex = (currentIndex - 1 + roles.length) % roles.length
        onRoleSelect(roles[prevIndex])
      }
    }

    window.addEventListener('keydown', handleKeyDown)
    return () => window.removeEventListener('keydown', handleKeyDown)
  }, [roles, selectedRole, onRoleSelect])
}

// ARIA labels
<div
  role="tree"
  aria-label="Role hierarchy"
  aria-activedescendant={selectedRole?.id}
>
  {/* Role tree nodes */}
</div>

// Screen reader announcements
const [announcement, setAnnouncement] = useState('')

useEffect(() => {
  if (selectedUser) {
    setAnnouncement(
      `Selected user: ${selectedUser.name}, ${selectedUser.roles.length} role(s)`
    )
  }
}, [selectedUser])

<div role="status" aria-live="polite" className="sr-only">
  {announcement}
</div>
```

---

## Performance

### Optimization Strategies

```typescript
/**
 * Performance targets:
 * - Initial load: <1s (with 1000 users, 50 roles)
 * - Role assignment: <500ms
 * - Search/filter: <300ms
 * - Bulk operation (100 users): <5s
 */

// 1. Virtualization for large user lists
import { useVirtualizer } from '@tanstack/react-virtual'

function VirtualizedUserList({ users }: { users: User[] }) {
  const parentRef = useRef<HTMLDivElement>(null)

  const virtualizer = useVirtualizer({
    count: users.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 60,
    overscan: 10,
  })

  return (
    <div ref={parentRef} className="h-96 overflow-auto">
      <div style={{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }}>
        {virtualizer.getVirtualItems().map((virtualItem) => {
          const user = users[virtualItem.index]
          return (
            <div
              key={virtualItem.key}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                height: `${virtualItem.size}px`,
                transform: `translateY(${virtualItem.start}px)`,
              }}
            >
              <UserRow user={user} />
            </div>
          )
        })}
      </div>
    </div>
  )
}

// 2. Debounced search
import { useDebouncedValue } from '@/hooks/useDebouncedValue'

const debouncedSearch = useDebouncedValue(searchQuery, props.searchDebounceMs || 300)

const usersQuery = useUsers({
  tenantId: tenantIdArray[0],
  search: debouncedSearch, // Not `searchQuery`
  filters,
  page: currentPage,
  pageSize: props.userPageSize || 50,
})

// 3. Memoized role hierarchy
const roleHierarchy = useMemo(
  () => buildRoleHierarchy(roles, props.roleHierarchy),
  [roles, props.roleHierarchy]
)

function buildRoleHierarchy(
  roles: Role[],
  hierarchyOverride?: Record<string, string[]>
): RoleHierarchyNode[] {
  // Build tree structure from flat role list
  const roleMap = new Map(roles.map((r) => [r.id, r]))
  const rootRoles: RoleHierarchyNode[] = []

  roles.forEach((role) => {
    const node: RoleHierarchyNode = { role, children: [] }

    if (role.inheritsFrom && role.inheritsFrom.length > 0) {
      // Has parent - will be added as child
    } else {
      // Root role
      rootRoles.push(node)
    }
  })

  // Add children
  roles.forEach((role) => {
    if (role.inheritsFrom) {
      role.inheritsFrom.forEach((parentId) => {
        const parent = findNodeById(rootRoles, parentId)
        if (parent) {
          parent.children.push({ role, children: [] })
        }
      })
    }
  })

  return rootRoles
}
```

---

## Multi-Domain Examples

### 1. Healthcare (HIPAA Role Separation)

```tsx
<RoleManager
  tenantId="healthcare-corp"
  currentUser={currentUser}
  availableRoles={[
    { id: 'physician', name: 'Physician', permissions: ['view_phi', 'edit_phi'] },
    { id: 'nurse', name: 'Nurse', permissions: ['view_phi'] },
    { id: 'admin', name: 'Admin', permissions: ['view_phi', 'audit_logs'] },
  ]}
  sensitiveRoles={['physician', 'admin']}
  requireApproval={true}
  requireChangeReason={true}
  dangerousPermissions={['view_phi', 'edit_phi']}
  onRoleAssign={(params) => {
    // Log HIPAA audit event
    auditLogger.logHIPAA({
      action: 'role_assign',
      ...params,
    })
  }}
/>
```

### 2. Financial Services (SOC 2 Compliance)

```tsx
<RoleManager
  tenantId="bank-xyz"
  currentUser={currentUser}
  enableTemporaryRoles={true}
  requireChangeReason={true}
  sensitiveRoles={['Auditor', 'Compliance_Officer', 'Admin']}
  requireApproval={true}
  onRoleAssign={(params) => {
    // Temporary roles for contractors
    if (params.expiresAt) {
      scheduleRoleRevocation({
        userId: params.userId,
        roleId: params.roleId,
        expiresAt: params.expiresAt,
      })
    }
  }}
/>
```

### 3. Legal Research (Attorney Roles)

```tsx
<RoleManager
  tenantId="law-firm-abc"
  currentUser={currentUser}
  roleTemplates={[
    { id: 'partner', name: 'Partner', roles: ['Admin', 'Attorney'] },
    { id: 'associate', name: 'Associate', roles: ['Attorney'] },
    { id: 'paralegal', name: 'Paralegal', roles: ['Viewer'] },
  ]}
  onRoleAssign={(params) => {
    // Check bar number for Attorney role
    if (params.roleId === 'Attorney') {
      const user = getUser(params.userId)
      if (!user.metadata?.barNumber) {
        throw new Error('Bar number required for Attorney role')
      }
    }
  }}
/>
```

### 4. Marketing Agency (Client Isolation)

```tsx
<RoleManager
  tenantId={['client-a', 'client-b', 'client-c']}
  currentUser={currentUser}
  layout="split"
  enableCrossTenantView={true}
  onRoleAssign={(params) => {
    // Ensure users only access their client's data
    const user = getUser(params.userId)
    if (user.tenantId !== params.tenantId) {
      throw new Error('Cannot assign cross-tenant role')
    }
  }}
/>
```

### 5. HR Platform (Department-Based Roles)

```tsx
<RoleManager
  tenantId="hr-platform"
  currentUser={currentUser}
  initialFilters={{
    departments: ['Engineering', 'Sales'],
  }}
  onRoleAssign={(params) => {
    // Auto-assign department-specific permissions
    const user = getUser(params.userId)
    const deptPermissions = getDepartmentPermissions(user.department)
    // Add dept permissions to role
  }}
/>
```

### 6. Academic Institution (Course-Based Access)

```tsx
<RoleManager
  tenantId="university"
  currentUser={currentUser}
  enableTemporaryRoles={true}
  roleTemplates={[
    { id: 'instructor', name: 'Instructor', roles: ['Instructor'] },
    { id: 'ta', name: 'Teaching Assistant', roles: ['TA'] },
    { id: 'student', name: 'Student', roles: ['Student'] },
  ]}
  onRoleAssign={(params) => {
    // Temporary roles for semester duration
    const semesterEnd = new Date('2025-12-15')
    return { ...params, expiresAt: semesterEnd }
  }}
/>
```

### 7. E-commerce (Customer Support Tiers)

```tsx
<RoleManager
  tenantId="ecommerce-store"
  currentUser={currentUser}
  roleHierarchy={{
    'Support_Manager': ['Support_L2', 'Support_L1'],
    'Support_L2': ['Support_L1'],
  }}
  enableBulkOperations={true}
  onRoleAssign={(params) => {
    // Track support tier changes
    analyticsService.track('support_tier_change', {
      userId: params.userId,
      newRole: params.roleId,
    })
  }}
/>
```

---

## Integration

### Integration with Objective Layer

```typescript
/**
 * Integrate RoleManager with OL primitives:
 * 1. User (OL) - Fetch user details
 * 2. Role (OL) - Fetch role definitions
 * 3. Permission (OL) - Fetch permission catalog
 * 4. AuditLog (OL) - Log role changes
 */

import { getUser, getRole, getPermissions, logAuditEvent } from '@/lib/ol'

async function fetchRolesFromOL(tenantId: string): Promise<Role[]> {
  const roles = await getRole({ tenantId })
  return roles.map((r) => ({
    id: r.id,
    name: r.name,
    displayName: r.display_name,
    description: r.description,
    permissions: r.permission_ids,
    inheritsFrom: r.parent_role_ids,
    system: r.is_system,
    tenantId: r.tenant_id,
    createdAt: new Date(r.created_at),
    updatedAt: new Date(r.updated_at),
    createdBy: r.created_by,
  }))
}

async function assignRoleViaOL(params: {
  userId: string
  roleId: string
  expiresAt?: Date
  reason?: string
}) {
  // 1. Fetch user and role from OL
  const user = await getUser(params.userId)
  const role = await getRole({ id: params.roleId })

  // 2. Assign role
  await assignRole({
    user_id: params.userId,
    role_id: params.roleId,
    expires_at: params.expiresAt?.toISOString(),
  })

  // 3. Log audit event
  await logAuditEvent({
    tenant_id: user.tenant_id,
    user_id: params.userId,
    action: 'role_assign',
    resource_type: 'role',
    resource_id: params.roleId,
    metadata: {
      role_name: role.name,
      expires_at: params.expiresAt?.toISOString(),
      reason: params.reason,
    },
  })
}
```

---

## Testing Strategy

### Unit Tests

```typescript
test('renders role hierarchy', () => {
  render(<RoleManager tenantId="test" currentUser={mockUser} />)
  expect(screen.getByText(/Role Hierarchy/i)).toBeInTheDocument()
})

test('filters users by role', async () => {
  render(<RoleManager tenantId="test" currentUser={mockUser} />)

  const roleFilter = screen.getByLabelText(/Filter by role/i)
  fireEvent.change(roleFilter, { target: { value: 'Admin' } })

  await waitFor(() => {
    expect(screen.queryByText(/Bob Smith/i)).not.toBeInTheDocument()
  })
})

test('assigns role to user', async () => {
  const onRoleAssign = jest.fn()
  render(<RoleManager tenantId="test" currentUser={mockUser} onRoleAssign={onRoleAssign} />)

  // Select user
  fireEvent.click(screen.getByText(/Alice Johnson/i))

  // Open assign dialog
  fireEvent.click(screen.getByText(/Assign Role/i))

  // Select role
  const roleSelect = screen.getByLabelText(/Select role/i)
  fireEvent.change(roleSelect, { target: { value: 'Editor' } })

  // Submit
  fireEvent.click(screen.getByText(/Assign/i))

  await waitFor(() => {
    expect(onRoleAssign).toHaveBeenCalledWith(
      expect.objectContaining({
        userId: 'alice',
        roleId: 'Editor',
      })
    )
  })
})
```

---

## References

- [Objective Layer Primitives](/docs/primitives/ol/)
- [SecurityDashboard Component](/docs/primitives/il/SecurityDashboard.md)
- [AuditViewer Component](/docs/primitives/il/AuditViewer.md)
- [TanStack Query](https://tanstack.com/query)
- [TanStack Virtual](https://tanstack.com/virtual)
- [RBAC Best Practices](https://en.wikipedia.org/wiki/Role-based_access_control)
