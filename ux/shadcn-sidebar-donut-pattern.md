# shadcn Sidebar with Next.js 16 Donut Pattern

**Date**: 2025-10-25
**Updated**: 2025-10-25 (After refactoring)
**Status**: ✅ Implemented
**Context**: Understanding and implementing shadcn sidebar using the donut pattern in Next.js 16

## Executive Summary

The donut pattern allows Server Components to be passed as children to Client Components, enabling data fetching on the server while maintaining client-side interactivity. For shadcn sidebar, this means:

- **Outer layer (Server)**: Wrapper that orchestrates the sidebar structure
- **Middle layer (Client)**: shadcn `Sidebar` components that handle interactivity (collapse state, hooks)
- **Inner layer (Server)**: Data-fetching components that render sidebar content

**Key Finding**: The original implementation passed data as props to a thick client component (194 lines). The refactored implementation uses the donut pattern with a thin client shell (~40 lines) that receives Server Components as children.

---

## Understanding the Donut Pattern

### Visual Representation

```
Server Component (outer - dough) 🍩
  ↓
Client Component (middle - ring - handles interactivity only)
  ↓
Server Component (inner - hole - fetches and renders data)
```

### Core Principle

**Instead of**: Server fetches data → passes as props → Client renders everything
**Use**: Server fetches data and renders → passes as children → Client provides interactivity

---

## Implementation Journey

### BEFORE: Original Implementation (NOT Donut Pattern)

**File structure:**
- `layout.tsx` - Server Component (sets up SidebarProvider)
- `program-sidebar.tsx` - Server Component (fetches data, prepares props)
- `program-sidebar-client.tsx` - Client Component (194 lines - renders EVERYTHING) ❌ REMOVED

**Flow:**
```typescript
// program-sidebar.tsx (Server)
export async function ProgramSidebar({ params }) {
  const { programId } = await params;
  const navItems = [...] // Prepare data
  const documents = [...] // Prepare data

  return (
    <ProgramSidebarClient
      programId={programId}
      programName={programName}
      navItems={navItems}      // ❌ Passing data as props
      documents={documents}    // ❌ Passing data as props
    />
  );
}

// program-sidebar-client.tsx (Client)
export function ProgramSidebarClient({ navItems, documents, ... }) {
  const { state, toggleSidebar } = useSidebar();

  return (
    <SidebarRoot>
      {/* ❌ Client component does ALL the rendering */}
      {navItems.map(item => ...)}
      {documents.map(doc => ...)}
    </SidebarRoot>
  );
}
```

**Why this was NOT the donut pattern:**
- `ProgramSidebarClient` was a **thick client component** (194 lines) doing all rendering logic
- Data was passed as **props** instead of Server Components as **children**
- Could not leverage server-side data fetching within the sidebar after initial render
- Lost the ability to use Suspense boundaries for individual sections

---

## AFTER: Refactored Donut Pattern Implementation

### File Structure (Actual Implementation)

**File structure:**
- `layout.tsx` - Server Component (sets up SidebarProvider)
- `program-sidebar.tsx` - Server Component (orchestrates structure, passes Server Components as children)
- `program-sidebar-shell.tsx` - Client Component (~40 lines - THIN wrapper, only handles interactivity)
- `sidebar-header.tsx` - Server Component (renders header content)
- `nav-items.tsx` - Server Component (renders navigation)
- `dashboards-collapsible.tsx` - Client Component (focused client component for collapsible)
- `documents-list.tsx` - async Server Component (fetches and renders documents)

### Actual Implementation Code

#### 1. Thin Client Shell (program-sidebar-shell.tsx)

**Purpose**: ONLY handles client-side interactivity
**Lines of code**: ~40 (vs 194 in the old thick client component)

```typescript
"use client";

import { ChevronLeft, ChevronRight } from "lucide-react";
import { Button } from "@/components/ui/button";
import { SidebarContent, SidebarHeader, Sidebar as SidebarRoot, useSidebar } from "@/components/ui/sidebar";

interface ProgramSidebarShellProps {
  header: React.ReactNode;      // Server Component
  navigation: React.ReactNode;  // Server Component
  documents: React.ReactNode;   // Server Component
}

export function ProgramSidebarShell({ header, navigation, documents }: ProgramSidebarShellProps) {
  const { state, toggleSidebar } = useSidebar();
  const isCollapsed = state === "collapsed";

  return (
    <SidebarRoot collapsible="icon" variant="sidebar">
      <SidebarHeader className="p-0">
        <div
          className={
            isCollapsed
              ? "flex items-center justify-center py-4"
              : "flex items-center justify-between gap-2 px-2 py-2 mb-2"
          }
        >
          {!isCollapsed && header}
          <Button variant="ghost" size="icon" className="h-10 w-10 shrink-0" onClick={toggleSidebar}>
            {isCollapsed ? <ChevronRight className="h-4 w-4" /> : <ChevronLeft className="h-4 w-4" />}
          </Button>
        </div>
      </SidebarHeader>

      <SidebarContent className={isCollapsed ? "items-center py-0" : "p-4"}>
        {navigation}
        {!isCollapsed && <div className="mt-6">{documents}</div>}
      </SidebarContent>
    </SidebarRoot>
  );
}
```

**Key characteristics:**
- ✅ Only handles client-side state (`useSidebar` hook)
- ✅ Only handles interactivity (toggle button, conditional CSS)
- ✅ Receives Server Components as `children` props
- ✅ Does NOT map over data or do rendering logic
- ✅ Thin: ~40 lines vs 194 lines in old version

#### 2. Server Component for Header (sidebar-header.tsx)

**Purpose**: Renders header content (can fetch data in the future)

```typescript
import { FileText } from "lucide-react";

interface SidebarHeaderContentProps {
  programName: string;
}

export function SidebarHeaderContent({ programName }: SidebarHeaderContentProps) {
  return (
    <div className="flex items-center gap-2 min-w-0 flex-1">
      <FileText className="h-4 w-4 text-muted-foreground shrink-0" />
      <span className="text-xs font-medium text-muted-foreground truncate">{programName}</span>
    </div>
  );
}
```

**Future enhancement:**
```typescript
export async function SidebarHeaderContent({ programId }: { programId: string }) {
  const program = await db.query.programs.findFirst({
    where: eq(programs.id, programId)
  });

  return (
    <div className="flex items-center gap-2 min-w-0 flex-1">
      <FileText className="h-4 w-4 text-muted-foreground shrink-0" />
      <span className="text-xs font-medium text-muted-foreground truncate">
        {program?.name ?? "Unknown Program"}
      </span>
    </div>
  );
}
```

#### 3. Server Component for Navigation (nav-items.tsx)

**Purpose**: Renders navigation menu items

```typescript
import { Database, LayoutDashboard, Users, Zap } from "lucide-react";
import Link from "next/link";
import { Badge } from "@/components/ui/badge";
import { SidebarGroup, SidebarMenu, SidebarMenuButton, SidebarMenuItem } from "@/components/ui/sidebar";
import { DashboardsCollapsible } from "./dashboards-collapsible";

type IconName = "LayoutDashboard" | "Users" | "Database";

interface NavItem {
  href: string;
  label: string;
  icon: IconName;
}

const iconMap = {
  LayoutDashboard,
  Users,
  Database,
};

function getIcon(iconName: IconName) {
  return iconMap[iconName];
}

interface NavItemsProps {
  programId: string;
  items: NavItem[];
}

export function NavItems({ programId, items }: NavItemsProps) {
  return (
    <SidebarGroup className="p-0">
      <SidebarMenu className="gap-1">
        {items.map((item) => {
          const Icon = getIcon(item.icon);
          return (
            <SidebarMenuItem key={item.href}>
              <SidebarMenuButton asChild tooltip={item.label}>
                <Link href={item.href}>
                  <Icon />
                  <span>{item.label}</span>
                </Link>
              </SidebarMenuButton>
            </SidebarMenuItem>
          );
        })}

        <DashboardsCollapsible programId={programId} />

        <SidebarMenuItem>
          <SidebarMenuButton tooltip="Deep Dive (BETA)">
            <Zap />
            <span>Deep Dive</span>
            <Badge variant="secondary" className="ml-auto text-xs">
              BETA
            </Badge>
          </SidebarMenuButton>
        </SidebarMenuItem>
      </SidebarMenu>
    </SidebarGroup>
  );
}
```

**Pattern used**: Server Component delegates interactive `Collapsible` to focused client component

#### 4. Client Component for Dashboards Collapsible (dashboards-collapsible.tsx)

**Purpose**: Small focused client component for the collapsible dashboards section

```typescript
"use client";

import { BarChart3, ChevronDown, Plus } from "lucide-react";
import Link from "next/link";
import { Collapsible, CollapsibleContent, CollapsibleTrigger } from "@/components/ui/collapsible";
import {
  SidebarMenuButton,
  SidebarMenuItem,
  SidebarMenuSub,
  SidebarMenuSubButton,
  SidebarMenuSubItem,
} from "@/components/ui/sidebar";

interface DashboardsCollapsibleProps {
  programId: string;
}

export function DashboardsCollapsible({ programId }: DashboardsCollapsibleProps) {
  return (
    <Collapsible defaultOpen className="group/collapsible">
      <SidebarMenuItem>
        <CollapsibleTrigger asChild>
          <SidebarMenuButton tooltip="Dashboards">
            <BarChart3 />
            <span>Dashboards</span>
            <ChevronDown className="ml-auto transition-transform group-data-[state=open]/collapsible:rotate-180" />
          </SidebarMenuButton>
        </CollapsibleTrigger>
        <CollapsibleContent>
          <SidebarMenuSub>
            <SidebarMenuSubItem>
              <SidebarMenuSubButton>
                <Plus className="h-3 w-3" />
                <span className="text-primary">New Dashboard</span>
              </SidebarMenuSubButton>
            </SidebarMenuSubItem>
            <SidebarMenuSubItem>
              <SidebarMenuSubButton asChild>
                <Link href={`/program/${programId}/user-dashboard`}>
                  <span>User Dashboard</span>
                </Link>
              </SidebarMenuSubButton>
            </SidebarMenuSubItem>
          </SidebarMenuSub>
        </CollapsibleContent>
      </SidebarMenuItem>
    </Collapsible>
  );
}
```

**Pattern**: Nested client component within Server Component for interactive elements

#### 5. async Server Component for Documents (documents-list.tsx)

**Purpose**: Fetches and renders documents (ready for database queries)

```typescript
import Link from "next/link";
import {
  SidebarGroup,
  SidebarGroupContent,
  SidebarGroupLabel,
  SidebarMenu,
  SidebarMenuButton,
  SidebarMenuItem,
} from "@/components/ui/sidebar";

interface Document {
  id: string;
  name: string;
}

export async function DocumentsList() {
  // This can now fetch data directly from the database
  // const documents = await db.query.documents.findMany({
  //   where: eq(documents.programId, programId)
  // });

  // For now, using static data
  const documents: Document[] = [
    { id: "wpvp-hazard-eval", name: "WPVP Hazard Identification & Evaluation" },
    { id: "searches", name: "Searches" },
    { id: "developer-questionnaire", name: "Developer AI Questionnaire" },
    { id: "incident-report", name: "Incident Report" },
  ];

  return (
    <SidebarGroup className="p-0">
      <SidebarGroupLabel className="px-2">Forms</SidebarGroupLabel>
      <SidebarGroupContent>
        <SidebarMenu className="gap-1">
          {documents.map((doc) => (
            <SidebarMenuItem key={doc.id}>
              <SidebarMenuButton asChild tooltip={doc.name} size="sm">
                <Link href={`/documents/${doc.id}`}>
                  <span className="truncate text-xs">{doc.name}</span>
                </Link>
              </SidebarMenuButton>
            </SidebarMenuItem>
          ))}
        </SidebarMenu>
      </SidebarGroupContent>
    </SidebarGroup>
  );
}
```

**Key benefit**: This component is `async`, meaning it can directly query the database in the future without any refactoring.

#### 6. Orchestrating Server Component (program-sidebar.tsx)

**Purpose**: Composes Server Components as children and wraps them in Suspense

```typescript
import { Suspense } from "react";
import { SidebarMenu, SidebarMenuItem, SidebarMenuSkeleton } from "@/components/ui/sidebar";
import { DocumentsList } from "./documents-list";
import { NavItems } from "./nav-items";
import { ProgramSidebarShell } from "./program-sidebar-shell";
import { SidebarHeaderContent } from "./sidebar-header";

interface ProgramSidebarProps {
  params: Promise<{ programId: string }>;
  programName?: string;
}

export async function ProgramSidebar({ params, programName = "Workplace Violence Prevention" }: ProgramSidebarProps) {
  "use cache";

  const { programId } = await params;

  // Define navigation items using programId
  const navItems = [
    { href: `/program/${programId}`, label: "Overview", icon: "LayoutDashboard" as const },
    { href: `/program/${programId}/assignment`, label: "Assignment", icon: "Users" as const },
    { href: `/program/${programId}/user-management`, label: "User Management", icon: "Users" as const },
    { href: `/program/${programId}/data-explorer`, label: "Data Explorer", icon: "Database" as const },
  ];

  return (
    <ProgramSidebarShell
      header={<SidebarHeaderContent programName={programName} />}
      navigation={
        <Suspense fallback={<NavItemsSkeleton />}>
          <NavItems programId={programId} items={navItems} />
        </Suspense>
      }
      documents={
        <Suspense fallback={<DocumentsSkeleton />}>
          <DocumentsList />
        </Suspense>
      }
    />
  );
}

function NavItemsSkeleton() {
  return (
    <SidebarMenu>
      {Array.from({ length: 5 }).map((_, i) => (
        // biome-ignore lint/suspicious/noArrayIndexKey: Static skeleton items won't reorder
        <SidebarMenuItem key={i}>
          <SidebarMenuSkeleton showIcon />
        </SidebarMenuItem>
      ))}
    </SidebarMenu>
  );
}

function DocumentsSkeleton() {
  return (
    <SidebarMenu>
      {Array.from({ length: 4 }).map((_, i) => (
        // biome-ignore lint/suspicious/noArrayIndexKey: Static skeleton items won't reorder
        <SidebarMenuItem key={i}>
          <SidebarMenuSkeleton />
        </SidebarMenuItem>
      ))}
    </SidebarMenu>
  );
}
```

**Key points:**
- ✅ Passes Server Components as children to the client shell
- ✅ Wraps each section in `Suspense` with skeleton loaders
- ✅ Uses `"use cache"` directive for caching
- ✅ Donut pattern: `ProgramSidebarShell` (client) receives `<SidebarHeaderContent />`, `<NavItems />`, and `<DocumentsList />` (server) as children

---

## Key Differences: Before vs After

| Aspect | BEFORE (Not Donut Pattern) | AFTER (Donut Pattern) |
|--------|---------------------------|----------------------|
| **Data Flow** | Server → props → Client | Server → children → Client |
| **Client Component Role** | Renders everything (194 lines) | Only handles interactivity (~40 lines) |
| **Server Component Role** | Prepares data only | Fetches AND renders data |
| **Data Fetching** | All upfront in wrapper | Can be distributed across components |
| **Suspense Boundaries** | Cannot use | Can wrap individual sections |
| **Collapsible Behavior** | Client handles all rendering logic | Client handles state, Server handles content |
| **Future DB Queries** | Would require refactoring | Already set up (`async` components) |

---

## Benefits Achieved

### 1. True Server-Side Data Fetching

```typescript
// Each section can fetch its own data independently
export async function DocumentsList() {
  const documents = await db.query.documents.findMany({
    where: eq(documents.programId, programId)
  });

  return <SidebarMenu>...</SidebarMenu>
}
```

### 2. Granular Suspense Boundaries

```typescript
<ProgramSidebarShell
  navigation={
    <Suspense fallback={<NavItemsSkeleton />}>
      <NavItems programId={programId} />
    </Suspense>
  }
  documents={
    <Suspense fallback={<DocumentsSkeleton />}>
      <DocumentsList />
    </Suspense>
  }
/>
```

### 3. Automatic Request Deduplication

Multiple Server Components can call the same data-fetching function without extra requests:

```typescript
// Both can call getProgram(programId) - Next.js deduplicates
async function SidebarHeaderContent({ programId }) {
  const program = await getProgram(programId);
  return <span>{program.name}</span>
}

async function NavItems({ programId }) {
  const program = await getProgram(programId); // Same request, cached
  const items = buildNavItems(program);
  return <SidebarMenu>...</SidebarMenu>
}
```

### 4. Better Code Organization

- ✅ Client components are thin and focused on interactivity
- ✅ Server components are focused on data and presentation
- ✅ Each section is self-contained and testable
- ✅ Easier to maintain and reason about

---

## Handling Collapsed State with Donut Pattern

### Challenge
The collapsed state is managed by the client component, but Server Components need to render differently when collapsed.

### Our Solution: CSS-Only (Implemented)

The client shell handles visibility with CSS and conditional rendering:

```typescript
// Client component (program-sidebar-shell.tsx)
export function ProgramSidebarShell({ header, navigation, documents }) {
  const { state } = useSidebar();
  const isCollapsed = state === "collapsed";

  return (
    <SidebarRoot>
      <SidebarContent className={isCollapsed ? "items-center py-0" : "p-4"}>
        {navigation}
        {/* Hide documents section when collapsed */}
        {!isCollapsed && <div className="mt-6">{documents}</div>}
      </SidebarContent>
    </SidebarRoot>
  );
}
```

**Benefits of this approach:**
- Simple and performant
- No prop drilling through Server Components
- Client component maintains full control over layout

---

## Handling Interactive Elements within Server Components

### Challenge
Some shadcn components like `Collapsible` require client-side state, but we want them inside Server Components.

### Our Solution: Nested Client Components (Implemented)

We extracted the interactive `Dashboards` section into a small focused client component (`dashboards-collapsible.tsx`), which is then imported into the Server Component:

```typescript
// nav-items.tsx (Server Component)
import { DashboardsCollapsible } from "./dashboards-collapsible";

export function NavItems({ programId, items }) {
  return (
    <SidebarMenu>
      {items.map(item => <SidebarMenuItem>...</SidebarMenuItem>)}

      {/* Delegate interactive collapsible to client component */}
      <DashboardsCollapsible programId={programId} />

      <SidebarMenuItem>Deep Dive</SidebarMenuItem>
    </SidebarMenu>
  );
}
```

**Pattern**: Server Component renders Server-only content → delegates interactive portions to small client components

---

## Conclusion

### What We Achieved

✅ **Refactored from thick client (194 lines) to thin client shell (~40 lines)**
✅ **Implemented true donut pattern**: Server Components as children, not data as props
✅ **Set up for future async data fetching** with `async` Server Components
✅ **Added granular Suspense boundaries** for better loading states
✅ **Improved code organization** with focused, single-responsibility components
✅ **Maintained full functionality** while improving architecture

### Files Created/Modified

**Created:**
- `program-sidebar-shell.tsx` - Thin client shell (~40 lines)
- `sidebar-header.tsx` - Server component for header
- `nav-items.tsx` - Server component for navigation
- `dashboards-collapsible.tsx` - Focused client component for collapsible
- `documents-list.tsx` - async Server component for documents

**Modified:**
- `program-sidebar.tsx` - Now orchestrates Server Components with Suspense

**Removed:**
- `program-sidebar-client.tsx` - Old thick client component (194 lines)

### The Donut Pattern in Practice

```
ProgramSidebar (Server - Orchestrator)
  ↓ passes Server Components as children to...
ProgramSidebarShell (Client - Thin Shell)
  ↓ renders Server Components...
<SidebarHeaderContent /> (Server)
<NavItems /> (Server)
  ↓ delegates interactive parts to...
  <DashboardsCollapsible /> (Client - Focused)
<DocumentsList /> (Server - async, ready for DB queries)
```

This pattern ensures the best of both worlds: server-side data fetching power with client-side interactivity, all while keeping components focused and maintainable.

---

## References

- Next.js 16 Best Practices: `docs/research/07-nextjs-16-best-practices.md:258-307` (Donut Pattern explanation)
- shadcn/ui Sidebar with RSC: Official shadcn documentation
- Next.js Server Components Composition: Official Next.js documentation
- Actual implementation: `src/app/(auth)/program/[programId]/_components/`
