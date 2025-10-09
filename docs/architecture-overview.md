# 🏗️ Architecture Overview

This document provides a high-level overview of Resource Tracker's architecture, technology choices, and design decisions.

## System Architecture

```mermaid
graph TB
    User[User Browser] --> Vercel[Vercel Edge Network]
    Vercel --> Next[Next.js App Router]
    Next --> Auth[NextAuth.js]
    Next --> API[API Routes]
    API --> Drizzle[Drizzle ORM]
    Drizzle --> Turso[(Turso SQLite)]
    Auth --> Discord[Discord OAuth]

    subgraph "Frontend"
        Next --> React[React Components]
        React --> Tailwind[Tailwind CSS]
        React --> Charts[Chart.js]
    end

    subgraph "Backend"
        API --> Middleware[Auth Middleware]
        API --> Validation[Input Validation]
        API --> Business[Business Logic]
    end
```

## Technology Stack

The Resource Tracker is built with a modern, TypeScript-first technology stack optimized for developer experience, type safety, and edge deployment.

| Component              | Technology       | Version   | Purpose                                           |
| ---------------------- | ---------------- | --------- | ------------------------------------------------- |
| **Frontend Framework** | Next.js          | `15.1.1`  | React-based framework with SSR & API routes       |
| **Language**           | TypeScript       | `5.9.3`   | Type-safe development and enhanced IDE support    |
| **Runtime**            | React            | `19.2.0`  | Component-based UI library                        |
| **Styling**            | Tailwind CSS     | `4.1.14`  | Utility-first CSS framework for responsive design |
| **Authentication**     | NextAuth.js      | `4.24.5`  | Authentication library with Discord provider      |
| **Database Client**    | `@libsql/client` | `0.15.15` | Turso SQLite database connectivity                |
| **ORM**                | Drizzle ORM      | `0.44.6`  | Type-safe database operations and migrations      |
| **Icons**              | Lucide React     | `0.544.0` | Modern icon library                               |
| **Unique IDs**         | nanoid           | `5.1.6`   | URL-safe unique ID generation                     |
| **Deployment**         | Vercel           | -         | Edge-optimized serverless platform                |

### Development Tools

| Tool                     | Version       | Purpose  |
| ------------------------ | ------------- | -------- | ----------------------------------------------- |
| **Database Migrations**  | `drizzle-kit` | `0.31.5` | Schema migration generation                     |
| **TypeScript Execution** | `tsx`         | `4.20.6` | High-performance TypeScript execution           |
| **Linting**              | ESLint        | `9.37.0` | Code quality and style enforcement              |
| **Environment Vars**     | `dotenv`      | `17.2.3` | Loading environment variables from `.env` files |

## Data Flow

### Authentication Flow

```
1. User clicks "Sign in with Discord"
2. Redirected to Discord OAuth
3. User authorizes application
4. Discord returns authorization code
5. NextAuth exchanges code for access token
6. Fetch user's Discord guild roles
7. Store session with role information
8. Redirect to dashboard
```

### Resource Update Flow

```
1. User submits a resource quantity change for a specific location (Hagga or Deep Desert).
2. Middleware validates the user's authentication session.
3. The API route checks if the user's role has the required permissions.
4. Input data (quantity, location, reason) is validated.
5. The status is recalculated based on the total quantity vs. the target.
6. The appropriate quantity field (`quantityHagga` or `quantityDeepDesert`) is updated in the `resources` table.
7. A detailed record is created in the `resource_history` table, capturing the before-and-after quantities for both locations.
8. Leaderboard points are calculated and stored.
9. The updated resource data is returned to the client.
10. The frontend UI updates optimistically to reflect the new state.
```

### Permission System

```
Role Configuration → Discord Roles → Session Storage → Route Protection

DISCORD_ROLES_CONFIG ──→ User's Discord Roles ──→ JWT Token ──→ API Authorization
                                                      ↓
                                              Middleware Check ──→ Allow/Deny
```

## Component Architecture

### Frontend Components

```
app/
├── layout.tsx              # Root layout with providers
├── page.tsx               # Landing page
├── components/            # Reusable components
│   ├── ClientNavigation.tsx   # Navigation with auth
│   ├── ResourceTable.tsx     # Main resource grid
│   ├── ResourceHistoryChart.tsx  # Time series charts
│   └── ThemeProvider.tsx     # Dark/light theme context
├── dashboard/             # Dashboard pages
└── resources/             # Resource management pages
```

### API Layer

```
app/api/
├── auth/[...nextauth]/    # NextAuth.js handler
├── resources/             # Resource CRUD operations
│   ├── route.ts              # List and create resources
│   ├── [id]/route.ts         # Get, update, delete resource
│   ├── [id]/history/         # Resource change history
│   └── [id]/target/          # Target quantity management
├── users/                 # User management endpoints
│   └── route.ts              # List all users (Admin)
├── user/                  # Endpoints for the authenticated user
│   ├── activity/route.ts     # User's personal activity
│   ├── data-export/route.ts  # GDPR data export for self
│   └── data-deletion/route.ts # GDPR data deletion for self
└── leaderboard/           # Points and rankings
```

## Database Design

### Schema Overview

The schema is designed around five core tables to handle users, resources, and their interactions.

- **`users`**: Stores essential information about a user, linked to their Discord ID.
- **`user_sessions`**: Caches user role data from Discord to reduce API calls and speed up authentication checks.
- **`resources`**: The central table for all trackable items. Crucially, it tracks quantities separately for two locations: `quantityHagga` and `quantityDeepDesert`.
- **`resource_history`**: Provides a detailed, immutable audit trail. Every change is recorded here, including the before-and-after quantities for _both_ locations to ensure complete traceability, even for transfers.
- **`leaderboard`**: Stores records of user contributions and calculated points for the gamification system.

### Key Relationships

- **Users → Resources**: Through `resource_history.updated_by`
- **Resources → History**: One-to-many relationship
- **Users → Leaderboard**: Points calculated from history
- **Resources → Status**: Calculated from total quantity vs. target

This architecture provides a solid foundation for a secure, scalable, and maintainable resource management system while remaining flexible enough for diverse organizational needs.
