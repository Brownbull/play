# OpenSaaS Development Guide - Complete Workflow Documentation

## Table of Contents
1. [Creating New Pages in the App](#1-creating-new-pages-in-the-app)
2. [Adding New Products to Sell](#2-adding-new-products-to-sell)
3. [Creating a Calendar View with Events](#3-creating-a-calendar-view-with-events)
4. [Creating New Database Entities](#4-creating-new-database-entities)
5. [TDD Approach Development](#5-tdd-approach-development)

---

## 1. Creating New Pages in the App

### Overview
In Wasp/OpenSaaS, pages are created by defining them in the `main.wasp` file and implementing the React component in the `src` directory.

### Step-by-Step Process

#### Step 1: Define Route and Page in main.wasp
```wasp
// main.wasp
route DashboardRoute {
  path: "/dashboard",
  to: DashboardPage
}

page DashboardPage {
  authRequired: true, // Set to true if authentication is required
  component: import { DashboardPage } from "@src/pages/Dashboard"
}
```

#### Step 2: Create the React Component
Create a new file `src/pages/Dashboard.tsx`:

```tsx
// src/pages/Dashboard.tsx
import React from 'react';
import { useAuth } from 'wasp/client/auth';

export const DashboardPage = () => {
  const { data: user } = useAuth();
  
  return (
    <div className="container mx-auto p-4">
      <h1 className="text-3xl font-bold mb-4">Dashboard</h1>
      <p>Welcome, {user?.email}!</p>
      {/* Add your page content here */}
    </div>
  );
};
```

#### Step 3: Add Navigation Link
Update your navigation component to include the new page:

```tsx
// src/components/Navigation.tsx
import { Link } from 'wasp/client/router';

<Link to="/dashboard" className="nav-link">
  Dashboard
</Link>
```

#### Step 4: Add Page with Parameters
For dynamic routes with parameters:

```wasp
// main.wasp
route UserProfileRoute {
  path: "/user/:userId",
  to: UserProfilePage
}

page UserProfilePage {
  authRequired: true,
  component: import { UserProfilePage } from "@src/pages/UserProfile"
}
```

```tsx
// src/pages/UserProfile.tsx
import { useParams } from 'react-router-dom';

export const UserProfilePage = () => {
  const { userId } = useParams<{ userId: string }>();
  
  return (
    <div>
      <h1>User Profile: {userId}</h1>
    </div>
  );
};
```

---

## 2. Adding New Products to Sell

### Overview
OpenSaaS uses Stripe for payment processing. Products are managed through Stripe's dashboard and referenced in your application.

### Step-by-Step Process

#### Step 1: Create Products in Stripe Dashboard
1. Go to [Stripe Dashboard](https://dashboard.stripe.com)
2. Navigate to **Products** → **+ Add Product**
3. Create your products with recurring prices:
   - Basic Plan: $10/month
   - Pro Plan: $25/month
   - Enterprise Plan: $50/month
4. Note the Price IDs (e.g., `price_1234567890`)

#### Step 2: Update Environment Variables
```bash
# .env.server
STRIPE_API_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Price IDs from Stripe
STRIPE_BASIC_PRICE_ID=price_basic123
STRIPE_PRO_PRICE_ID=price_pro456
STRIPE_ENTERPRISE_PRICE_ID=price_enterprise789
```

#### Step 3: Define Products in Application
Create a products configuration file:

```typescript
// src/shared/constants.ts
export const PRODUCTS = {
  basic: {
    name: 'Basic',
    priceId: process.env.STRIPE_BASIC_PRICE_ID,
    price: 10,
    features: [
      '10 Projects',
      'Basic Support',
      '1 Team Member'
    ]
  },
  pro: {
    name: 'Pro',
    priceId: process.env.STRIPE_PRO_PRICE_ID,
    price: 25,
    features: [
      'Unlimited Projects',
      'Priority Support',
      '5 Team Members',
      'Advanced Analytics'
    ]
  },
  enterprise: {
    name: 'Enterprise',
    priceId: process.env.STRIPE_ENTERPRISE_PRICE_ID,
    price: 50,
    features: [
      'Everything in Pro',
      'Unlimited Team Members',
      'Custom Integrations',
      'Dedicated Support'
    ]
  }
};
```

#### Step 4: Create Pricing Page Component
```tsx
// src/pages/Pricing.tsx
import React from 'react';
import { PRODUCTS } from '@src/shared/constants';
import { createCheckoutSession } from '@src/operations/stripe';

export const PricingPage = () => {
  const handleSubscribe = async (priceId: string) => {
    try {
      const session = await createCheckoutSession({ priceId });
      window.location.href = session.url;
    } catch (error) {
      console.error('Error creating checkout session:', error);
    }
  };

  return (
    <div className="container mx-auto p-8">
      <h1 className="text-4xl font-bold text-center mb-8">Pricing</h1>
      <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
        {Object.entries(PRODUCTS).map(([key, product]) => (
          <div key={key} className="border rounded-lg p-6">
            <h2 className="text-2xl font-bold mb-4">{product.name}</h2>
            <p className="text-3xl font-bold mb-6">${product.price}/mo</p>
            <ul className="mb-6">
              {product.features.map((feature, index) => (
                <li key={index} className="mb-2">✓ {feature}</li>
              ))}
            </ul>
            <button
              onClick={() => handleSubscribe(product.priceId)}
              className="w-full bg-blue-600 text-white py-2 rounded hover:bg-blue-700"
            >
              Subscribe
            </button>
          </div>
        ))}
      </div>
    </div>
  );
};
```

#### Step 5: Create Stripe Operations
```typescript
// src/operations/stripe.ts
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_API_KEY!, {
  apiVersion: '2023-10-16'
});

export const createCheckoutSession = async (args: { priceId: string }, context: any) => {
  const user = context.user;
  
  if (!user) {
    throw new Error('User must be logged in');
  }

  const session = await stripe.checkout.sessions.create({
    customer_email: user.email,
    payment_method_types: ['card'],
    line_items: [
      {
        price: args.priceId,
        quantity: 1,
      },
    ],
    mode: 'subscription',
    success_url: `${process.env.APP_URL}/dashboard?success=true`,
    cancel_url: `${process.env.APP_URL}/pricing?canceled=true`,
    metadata: {
      userId: user.id.toString(),
    },
  });

  return { url: session.url };
};
```

---

## 3. Creating a Calendar View with Events

### Overview
Implement a calendar view with event management using React Big Calendar or FullCalendar.

### Step-by-Step Process

#### Step 1: Install Calendar Library
```bash
npm install react-big-calendar moment
# or
npm install @fullcalendar/react @fullcalendar/daygrid @fullcalendar/interaction
```

#### Step 2: Define Event Entity in Prisma Schema
```prisma
// schema.prisma
model CalendarEvent {
  id          Int      @id @default(autoincrement())
  title       String
  description String?
  start       DateTime
  end         DateTime
  allDay      Boolean  @default(false)
  userId      Int
  user        User     @relation(fields: [userId], references: [id])
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}

model User {
  // ... existing fields
  events      CalendarEvent[]
}
```

#### Step 3: Run Migration
```bash
wasp db migrate-dev
```

#### Step 4: Create Calendar Operations in main.wasp
```wasp
// main.wasp
query getUserEvents {
  fn: import { getUserEvents } from "@src/queries/calendar",
  entities: [CalendarEvent]
}

action createEvent {
  fn: import { createEvent } from "@src/actions/calendar",
  entities: [CalendarEvent]
}

action updateEvent {
  fn: import { updateEvent } from "@src/actions/calendar",
  entities: [CalendarEvent]
}

action deleteEvent {
  fn: import { deleteEvent } from "@src/actions/calendar",
  entities: [CalendarEvent]
}
```

#### Step 5: Implement Query and Actions
```typescript
// src/queries/calendar.ts
import { CalendarEvent } from 'wasp/entities';
import { GetUserEvents } from 'wasp/server/operations';

export const getUserEvents: GetUserEvents<void, CalendarEvent[]> = async (args, context) => {
  if (!context.user) {
    throw new Error('User not authenticated');
  }

  return context.entities.CalendarEvent.findMany({
    where: { userId: context.user.id },
    orderBy: { start: 'asc' }
  });
};
```

```typescript
// src/actions/calendar.ts
import { CalendarEvent } from 'wasp/entities';
import { CreateEvent, UpdateEvent, DeleteEvent } from 'wasp/server/operations';

type CreateEventInput = {
  title: string;
  description?: string;
  start: Date;
  end: Date;
  allDay?: boolean;
};

export const createEvent: CreateEvent<CreateEventInput, CalendarEvent> = async (args, context) => {
  if (!context.user) {
    throw new Error('User not authenticated');
  }

  return context.entities.CalendarEvent.create({
    data: {
      ...args,
      userId: context.user.id
    }
  });
};

export const updateEvent: UpdateEvent<{ id: number } & Partial<CreateEventInput>, CalendarEvent> = async (args, context) => {
  if (!context.user) {
    throw new Error('User not authenticated');
  }

  const { id, ...data } = args;

  return context.entities.CalendarEvent.update({
    where: { id, userId: context.user.id },
    data
  });
};

export const deleteEvent: DeleteEvent<{ id: number }, void> = async (args, context) => {
  if (!context.user) {
    throw new Error('User not authenticated');
  }

  await context.entities.CalendarEvent.delete({
    where: { id: args.id, userId: context.user.id }
  });
};
```

#### Step 6: Create Calendar Component
```tsx
// src/pages/Calendar.tsx
import React, { useState, useCallback } from 'react';
import { Calendar, momentLocalizer, View } from 'react-big-calendar';
import moment from 'moment';
import 'react-big-calendar/lib/css/react-big-calendar.css';
import { useQuery, useAction } from 'wasp/client/operations';
import { getUserEvents, createEvent, updateEvent, deleteEvent } from 'wasp/client/operations';

const localizer = momentLocalizer(moment);

export const CalendarPage = () => {
  const { data: events, isLoading } = useQuery(getUserEvents);
  const createEventFn = useAction(createEvent);
  const updateEventFn = useAction(updateEvent);
  const deleteEventFn = useAction(deleteEvent);
  
  const [view, setView] = useState<View>('month');
  const [date, setDate] = useState(new Date());
  const [showModal, setShowModal] = useState(false);
  const [selectedEvent, setSelectedEvent] = useState<any>(null);

  const handleSelectSlot = useCallback(({ start, end }: any) => {
    const title = window.prompt('Enter event title:');
    if (title) {
      createEventFn({
        title,
        start,
        end,
        allDay: false
      });
    }
  }, [createEventFn]);

  const handleSelectEvent = useCallback((event: any) => {
    setSelectedEvent(event);
    setShowModal(true);
  }, []);

  const handleEventDrop = useCallback(({ event, start, end }: any) => {
    updateEventFn({
      id: event.id,
      start,
      end
    });
  }, [updateEventFn]);

  const calendarEvents = events?.map(event => ({
    ...event,
    start: new Date(event.start),
    end: new Date(event.end)
  })) || [];

  if (isLoading) return <div>Loading...</div>;

  return (
    <div className="h-screen p-4">
      <h1 className="text-3xl font-bold mb-4">Calendar</h1>
      <Calendar
        localizer={localizer}
        events={calendarEvents}
        startAccessor="start"
        endAccessor="end"
        style={{ height: 'calc(100vh - 120px)' }}
        view={view}
        onView={setView}
        date={date}
        onNavigate={setDate}
        onSelectSlot={handleSelectSlot}
        onSelectEvent={handleSelectEvent}
        onEventDrop={handleEventDrop}
        selectable
        resizable
        draggableAccessor={() => true}
      />
      
      {/* Event Modal */}
      {showModal && selectedEvent && (
        <EventModal
          event={selectedEvent}
          onClose={() => setShowModal(false)}
          onUpdate={updateEventFn}
          onDelete={deleteEventFn}
        />
      )}
    </div>
  );
};

// Event Modal Component
const EventModal = ({ event, onClose, onUpdate, onDelete }: any) => {
  const [title, setTitle] = useState(event.title);
  const [description, setDescription] = useState(event.description || '');

  const handleUpdate = () => {
    onUpdate({
      id: event.id,
      title,
      description
    });
    onClose();
  };

  const handleDelete = () => {
    if (window.confirm('Delete this event?')) {
      onDelete({ id: event.id });
      onClose();
    }
  };

  return (
    <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center">
      <div className="bg-white p-6 rounded-lg w-96">
        <h2 className="text-xl font-bold mb-4">Edit Event</h2>
        <input
          type="text"
          value={title}
          onChange={(e) => setTitle(e.target.value)}
          className="w-full p-2 border rounded mb-4"
          placeholder="Event title"
        />
        <textarea
          value={description}
          onChange={(e) => setDescription(e.target.value)}
          className="w-full p-2 border rounded mb-4"
          placeholder="Description"
          rows={3}
        />
        <div className="flex justify-between">
          <button
            onClick={handleDelete}
            className="px-4 py-2 bg-red-500 text-white rounded"
          >
            Delete
          </button>
          <div>
            <button
              onClick={onClose}
              className="px-4 py-2 bg-gray-300 rounded mr-2"
            >
              Cancel
            </button>
            <button
              onClick={handleUpdate}
              className="px-4 py-2 bg-blue-500 text-white rounded"
            >
              Save
            </button>
          </div>
        </div>
      </div>
    </div>
  );
};
```

---

## 4. Creating New Database Entities

### Overview
Wasp uses Prisma for database management. Entities are defined in the `schema.prisma` file and accessed through Wasp operations.

### Step-by-Step Process

#### Step 1: Define Entity in Prisma Schema
```prisma
// schema.prisma
model UserProfile {
  id              Int      @id @default(autoincrement())
  userId          Int      @unique
  user            User     @relation(fields: [userId], references: [id])
  firstName       String?
  lastName        String?
  bio             String?
  avatarUrl       String?
  phoneNumber     String?
  dateOfBirth     DateTime?
  address         String?
  city            String?
  country         String?
  zipCode         String?
  linkedinUrl     String?
  twitterHandle   String?
  websiteUrl      String?
  skills          String[] // Array of skills
  preferences     Json?    // JSON field for flexible data
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
}

// Add relation to User model
model User {
  // ... existing fields
  profile         UserProfile?
}
```

#### Step 2: Run Database Migration
```bash
# Stop the dev server if running
# Run migration
wasp db migrate-dev

# Name your migration (e.g., "add_user_profile")
```

#### Step 3: Define Operations in main.wasp
```wasp
// main.wasp
query getUserProfile {
  fn: import { getUserProfile } from "@src/queries/userProfile",
  entities: [UserProfile, User]
}

action createUserProfile {
  fn: import { createUserProfile } from "@src/actions/userProfile",
  entities: [UserProfile, User]
}

action updateUserProfile {
  fn: import { updateUserProfile } from "@src/actions/userProfile",
  entities: [UserProfile]
}
```

#### Step 4: Implement Queries
```typescript
// src/queries/userProfile.ts
import { UserProfile } from 'wasp/entities';
import { GetUserProfile } from 'wasp/server/operations';

export const getUserProfile: GetUserProfile<void, UserProfile | null> = async (args, context) => {
  if (!context.user) {
    throw new Error('User not authenticated');
  }

  const profile = await context.entities.UserProfile.findUnique({
    where: { userId: context.user.id },
    include: {
      user: {
        select: {
          email: true,
          username: true
        }
      }
    }
  });

  return profile;
};
```

#### Step 5: Implement Actions
```typescript
// src/actions/userProfile.ts
import { UserProfile } from 'wasp/entities';
import { CreateUserProfile, UpdateUserProfile } from 'wasp/server/operations';

type ProfileInput = {
  firstName?: string;
  lastName?: string;
  bio?: string;
  phoneNumber?: string;
  city?: string;
  country?: string;
  skills?: string[];
  preferences?: any;
};

export const createUserProfile: CreateUserProfile<ProfileInput, UserProfile> = async (args, context) => {
  if (!context.user) {
    throw new Error('User not authenticated');
  }

  // Check if profile already exists
  const existing = await context.entities.UserProfile.findUnique({
    where: { userId: context.user.id }
  });

  if (existing) {
    throw new Error('Profile already exists');
  }

  return context.entities.UserProfile.create({
    data: {
      ...args,
      userId: context.user.id
    }
  });
};

export const updateUserProfile: UpdateUserProfile<ProfileInput, UserProfile> = async (args, context) => {
  if (!context.user) {
    throw new Error('User not authenticated');
  }

  return context.entities.UserProfile.update({
    where: { userId: context.user.id },
    data: args
  });
};
```

#### Step 6: Create Profile Component
```tsx
// src/pages/Profile.tsx
import React, { useState, useEffect } from 'react';
import { useQuery, useAction } from 'wasp/client/operations';
import { getUserProfile, createUserProfile, updateUserProfile } from 'wasp/client/operations';

export const ProfilePage = () => {
  const { data: profile, isLoading } = useQuery(getUserProfile);
  const createProfile = useAction(createUserProfile);
  const updateProfile = useAction(updateUserProfile);
  
  const [formData, setFormData] = useState({
    firstName: '',
    lastName: '',
    bio: '',
    phoneNumber: '',
    city: '',
    country: '',
    skills: [] as string[]
  });

  useEffect(() => {
    if (profile) {
      setFormData({
        firstName: profile.firstName || '',
        lastName: profile.lastName || '',
        bio: profile.bio || '',
        phoneNumber: profile.phoneNumber || '',
        city: profile.city || '',
        country: profile.country || '',
        skills: profile.skills || []
      });
    }
  }, [profile]);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    
    try {
      if (profile) {
        await updateProfile(formData);
      } else {
        await createProfile(formData);
      }
      alert('Profile saved successfully!');
    } catch (error) {
      console.error('Error saving profile:', error);
      alert('Error saving profile');
    }
  };

  if (isLoading) return <div>Loading...</div>;

  return (
    <div className="container mx-auto p-4 max-w-2xl">
      <h1 className="text-3xl font-bold mb-6">User Profile</h1>
      
      <form onSubmit={handleSubmit} className="space-y-4">
        <div className="grid grid-cols-2 gap-4">
          <div>
            <label className="block text-sm font-medium mb-1">First Name</label>
            <input
              type="text"
              value={formData.firstName}
              onChange={(e) => setFormData({ ...formData, firstName: e.target.value })}
              className="w-full p-2 border rounded"
            />
          </div>
          <div>
            <label className="block text-sm font-medium mb-1">Last Name</label>
            <input
              type="text"
              value={formData.lastName}
              onChange={(e) => setFormData({ ...formData, lastName: e.target.value })}
              className="w-full p-2 border rounded"
            />
          </div>
        </div>

        <div>
          <label className="block text-sm font-medium mb-1">Bio</label>
          <textarea
            value={formData.bio}
            onChange={(e) => setFormData({ ...formData, bio: e.target.value })}
            className="w-full p-2 border rounded"
            rows={4}
          />
        </div>

        <div className="grid grid-cols-2 gap-4">
          <div>
            <label className="block text-sm font-medium mb-1">Phone Number</label>
            <input
              type="tel"
              value={formData.phoneNumber}
              onChange={(e) => setFormData({ ...formData, phoneNumber: e.target.value })}
              className="w-full p-2 border rounded"
            />
          </div>
          <div>
            <label className="block text-sm font-medium mb-1">City</label>
            <input
              type="text"
              value={formData.city}
              onChange={(e) => setFormData({ ...formData, city: e.target.value })}
              className="w-full p-2 border rounded"
            />
          </div>
        </div>

        <div>
          <label className="block text-sm font-medium mb-1">Country</label>
          <input
            type="text"
            value={formData.country}
            onChange={(e) => setFormData({ ...formData, country: e.target.value })}
            className="w-full p-2 border rounded"
          />
        </div>

        <button
          type="submit"
          className="w-full bg-blue-600 text-white py-2 rounded hover:bg-blue-700"
        >
          {profile ? 'Update Profile' : 'Create Profile'}
        </button>
      </form>
    </div>
  );
};
```

#### Step 7: Advanced Entity Relationships
For more complex relationships:

```prisma
// schema.prisma - Many-to-Many relationship example
model Project {
  id          Int                @id @default(autoincrement())
  name        String
  description String?
  ownerId     Int
  owner       User               @relation(fields: [ownerId], references: [id])
  members     ProjectMember[]
  tasks       Task[]
  createdAt   DateTime           @default(now())
  updatedAt   DateTime           @updatedAt
}

model ProjectMember {
  id        Int      @id @default(autoincrement())
  projectId Int
  project   Project  @relation(fields: [projectId], references: [id])
  userId    Int
  user      User     @relation(fields: [userId], references: [id])
  role      String   @default("member") // "owner", "admin", "member"
  joinedAt  DateTime @default(now())
  
  @@unique([projectId, userId])
}

model Task {
  id          Int      @id @default(autoincrement())
  title       String
  description String?
  projectId   Int
  project     Project  @relation(fields: [projectId], references: [id])
  assigneeId  Int?
  assignee    User?    @relation(fields: [assigneeId], references: [id])
  status      String   @default("todo") // "todo", "in_progress", "done"
  priority    String   @default("medium") // "low", "medium", "high"
  dueDate     DateTime?
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}

model User {
  // ... existing fields
  ownedProjects Project[]       @relation("ProjectOwner")
  projectMemberships ProjectMember[]
  assignedTasks Task[]
}
```

---

## 5. TDD Approach Development

### Overview
Test-Driven Development with Wasp using Vitest for unit tests and Playwright for E2E tests.

### Step-by-Step Process

#### Step 1: Setup Testing Environment
```bash
# Install testing dependencies
npm install -D vitest @vitest/ui @testing-library/react @testing-library/jest-dom jsdom
npm install -D @playwright/test

# For component testing
npm install -D @testing-library/user-event
```

#### Step 2: Configure Vitest
Create `vitest.config.ts`:

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: './src/test/setup.ts',
    coverage: {
      reporter: ['text', 'json', 'html'],
      exclude: [
        'node_modules/',
        'src/test/',
      ],
    },
  },
  resolve: {
    alias: {
      '@src': path.resolve(__dirname, './src'),
      'wasp': path.resolve(__dirname, './src/test/mocks/wasp'),
    },
  },
});
```

#### Step 3: Setup Test Configuration
```typescript
// src/test/setup.ts
import '@testing-library/jest-dom';
import { expect, afterEach } from 'vitest';
import { cleanup } from '@testing-library/react';
import * as matchers from '@testing-library/jest-dom/matchers';

expect.extend(matchers);

afterEach(() => {
  cleanup();
});
```

#### Step 4: Create Mock Utilities
```typescript
// src/test/mocks/wasp.ts
import { vi } from 'vitest';

export const useQuery = vi.fn();
export const useAction = vi.fn();
export const useAuth = vi.fn();

export const client = {
  operations: {
    useQuery,
    useAction,
  },
  auth: {
    useAuth,
  },
};
```

#### Step 5: TDD Workflow Example - Creating a Todo Feature

##### 5.1: Write the Test First (RED Phase)
```typescript
// src/features/todo/__tests__/TodoList.test.tsx
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { TodoList } from '../TodoList';

describe('TodoList Component', () => {
  const mockTodos = [
    { id: 1, title: 'Test Todo 1', completed: false },
    { id: 2, title: 'Test Todo 2', completed: true },
  ];

  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('should display a list of todos', () => {
    render(<TodoList todos={mockTodos} />);
    
    expect(screen.getByText('Test Todo 1')).toBeInTheDocument();
    expect(screen.getByText('Test Todo 2')).toBeInTheDocument();
  });

  it('should add a new todo when form is submitted', async () => {
    const onAddTodo = vi.fn();
    const user = userEvent.setup();
    
    render(<TodoList todos={mockTodos} onAddTodo={onAddTodo} />);
    
    const input = screen.getByPlaceholderText('Add a new todo');
    const button = screen.getByText('Add');
    
    await user.type(input, 'New Todo');
    await user.click(button);
    
    expect(onAddTodo).toHaveBeenCalledWith('New Todo');
  });

  it('should toggle todo completion status', async () => {
    const onToggleTodo = vi.fn();
    const user = userEvent.setup();
    
    render(<TodoList todos={mockTodos} onToggleTodo={onToggleTodo} />);
    
    const checkbox = screen.getAllByRole('checkbox')[0];
    await user.click(checkbox);
    
    expect(onToggleTodo).toHaveBeenCalledWith(1);
  });

  it('should delete a todo', async () => {
    const onDeleteTodo = vi.fn();
    const user = userEvent.setup();
    
    render(<TodoList todos={mockTodos} onDeleteTodo={onDeleteTodo} />);
    
    const deleteButtons = screen.getAllByText('Delete');
    await user.click(deleteButtons[0]);
    
    expect(onDeleteTodo).toHaveBeenCalledWith(1);
  });

  it('should filter todos by status', async () => {
    const user = userEvent.setup();
    
    render(<TodoList todos={mockTodos} />);
    
    // Show only active todos
    const activeFilter = screen.getByText('Active');
    await user.click(activeFilter);
    
    expect(screen.getByText('Test Todo 1')).toBeInTheDocument();
    expect(screen.queryByText('Test Todo 2')).not.toBeInTheDocument();
    
    // Show only completed todos
    const completedFilter = screen.getByText('Completed');
    await user.click(completedFilter);
    
    expect(screen.queryByText('Test Todo 1')).not.toBeInTheDocument();
    expect(screen.getByText('Test Todo 2')).toBeInTheDocument();
  });
});
```

##### 5.2: Write Minimal Code to Pass (GREEN Phase)
```tsx
// src/features/todo/TodoList.tsx
import React, { useState } from 'react';

interface Todo {
  id: number;
  title: string;
  completed: boolean;
}

interface TodoListProps {
  todos: Todo[];
  onAddTodo?: (title: string) => void;
  onToggleTodo?: (id: number) => void;
  onDeleteTodo?: (id: number) => void;
}

export const TodoList: React.FC<TodoListProps> = ({
  todos,
  onAddTodo,
  onToggleTodo,
  onDeleteTodo,
}) => {
  const [newTodo, setNewTodo] = useState('');
  const [filter, setFilter] = useState<'all' | 'active' | 'completed'>('all');

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (newTodo.trim() && onAddTodo) {
      onAddTodo(newTodo.trim());
      setNewTodo('');
    }
  };

  const filteredTodos = todos.filter(todo => {
    if (filter === 'active') return !todo.completed;
    if (filter === 'completed') return todo.completed;
    return true;
  });

  return (
    <div className="todo-list">
      <form onSubmit={handleSubmit} className="mb-4">
        <input
          type="text"
          value={newTodo}
          onChange={(e) => setNewTodo(e.target.value)}
          placeholder="Add a new todo"
          className="p-2 border rounded mr-2"
        />
        <button type="submit" className="px-4 py-2 bg-blue-500 text-white rounded">
          Add
        </button>
      </form>

      <div className="filters mb-4">
        <button onClick={() => setFilter('all')}>All</button>
        <button onClick={() => setFilter('active')}>Active</button>
        <button onClick={() => setFilter('completed')}>Completed</button>
      </div>

      <ul className="space-y-2">
        {filteredTodos.map(todo => (
          <li key={todo.id} className="flex items-center">
            <input
              type="checkbox"
              checked={todo.completed}
              onChange={() => onToggleTodo?.(todo.id)}
              className="mr-2"
            />
            <span className={todo.completed ? 'line-through' : ''}>
              {todo.title}
            </span>
            <button
              onClick={() => onDeleteTodo?.(todo.id)}
              className="ml-auto px-2 py-1 bg-red-500 text-white rounded"
            >
              Delete
            </button>
          </li>
        ))}
      </ul>
    </div>
  );
};
```

##### 5.3: Refactor (REFACTOR Phase)
```tsx
// src/features/todo/TodoList.tsx (Refactored)
import React, { useState, useMemo } from 'react';
import { TodoItem } from './TodoItem';
import { TodoForm } from './TodoForm';
import { TodoFilters } from './TodoFilters';
import { Todo, FilterType } from './types';

interface TodoListProps {
  todos: Todo[];
  onAddTodo?: (title: string) => void;
  onToggleTodo?: (id: number) => void;
  onDeleteTodo?: (id: number) => void;
}

export const TodoList: React.FC<TodoListProps> = ({
  todos,
  onAddTodo,
  onToggleTodo,
  onDeleteTodo,
}) => {
  const [filter, setFilter] = useState<FilterType>('all');

  const filteredTodos = useMemo(() => {
    switch (filter) {
      case 'active':
        return todos.filter(todo => !todo.completed);
      case 'completed':
        return todos.filter(todo => todo.completed);
      default:
        return todos;
    }
  }, [todos, filter]);

  const stats = useMemo(() => ({
    total: todos.length,
    active: todos.filter(t => !t.completed).length,
    completed: todos.filter(t => t.completed).length,
  }), [todos]);

  return (
    <div className="todo-list max-w-2xl mx-auto p-4">
      <h1 className="text-2xl font-bold mb-4">Todo List</h1>
      
      <TodoForm onSubmit={onAddTodo} />
      
      <TodoFilters 
        currentFilter={filter} 
        onFilterChange={setFilter}
        stats={stats}
      />

      <ul className="space-y-2">
        {filteredTodos.map(todo => (
          <TodoItem
            key={todo.id}
            todo={todo}
            onToggle={() => onToggleTodo?.(todo.id)}
            onDelete={() => onDeleteTodo?.(todo.id)}
          />
        ))}
      </ul>

      {filteredTodos.length === 0 && (
        <p className="text-gray-500 text-center mt-4">
          No todos to display
        </p>
      )}
    </div>
  );
};
```

#### Step 6: Integration Tests
```typescript
// src/features/todo/__tests__/TodoIntegration.test.tsx
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { TodoPage } from '../TodoPage';
import * as todoOperations from '../operations';

// Mock the operations
vi.mock('../operations', () => ({
  useGetTodos: vi.fn(),
  useCreateTodo: vi.fn(),
  useUpdateTodo: vi.fn(),
  useDeleteTodo: vi.fn(),
}));

describe('Todo Page Integration', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('should load and display todos from the server', async () => {
    const mockTodos = [
      { id: 1, title: 'Server Todo 1', completed: false },
      { id: 2, title: 'Server Todo 2', completed: true },
    ];

    todoOperations.useGetTodos.mockReturnValue({
      data: mockTodos,
      isLoading: false,
      error: null,
    });

    todoOperations.useCreateTodo.mockReturnValue({
      mutate: vi.fn(),
      isLoading: false,
    });

    render(<TodoPage />);

    await waitFor(() => {
      expect(screen.getByText('Server Todo 1')).toBeInTheDocument();
      expect(screen.getByText('Server Todo 2')).toBeInTheDocument();
    });
  });

  it('should create a new todo and update the list', async () => {
    const user = userEvent.setup();
    const mockCreate = vi.fn();

    todoOperations.useGetTodos.mockReturnValue({
      data: [],
      isLoading: false,
      error: null,
    });

    todoOperations.useCreateTodo.mockReturnValue({
      mutate: mockCreate,
      isLoading: false,
    });

    render(<TodoPage />);

    const input = screen.getByPlaceholderText('What needs to be done?');
    const button = screen.getByText('Add Todo');

    await user.type(input, 'New Integration Todo');
    await user.click(button);

    expect(mockCreate).toHaveBeenCalledWith({
      title: 'New Integration Todo',
    });
  });
});
```

#### Step 7: E2E Tests with Playwright
```typescript
// e2e/todo.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Todo Application E2E', () => {
  test.beforeEach(async ({ page }) => {
    // Login before each test
    await page.goto('/login');
    await page.fill('[name="email"]', 'test@example.com');
    await page.fill('[name="password"]', 'testpassword');
    await page.click('button[type="submit"]');
    
    // Wait for redirect to todo page
    await page.waitForURL('/todos');
  });

  test('should create, complete, and delete a todo', async ({ page }) => {
    // Create a new todo
    const todoTitle = `E2E Todo ${Date.now()}`;
    await page.fill('[placeholder="What needs to be done?"]', todoTitle);
    await page.click('button:has-text("Add Todo")');

    // Verify todo appears
    await expect(page.locator(`text=${todoTitle}`)).toBeVisible();

    // Complete the todo
    await page.click(`[data-testid="todo-checkbox-${todoTitle}"]`);
    await expect(page.locator(`[data-testid="todo-item-${todoTitle}"]`))
      .toHaveClass(/completed/);

    // Delete the todo
    await page.click(`[data-testid="delete-${todoTitle}"]`);
    await expect(page.locator(`text=${todoTitle}`)).not.toBeVisible();
  });

  test('should filter todos correctly', async ({ page }) => {
    // Create multiple todos
    await page.fill('[placeholder="What needs to be done?"]', 'Active Todo');
    await page.click('button:has-text("Add Todo")');
    
    await page.fill('[placeholder="What needs to be done?"]', 'Completed Todo');
    await page.click('button:has-text("Add Todo")');
    
    // Complete one todo
    await page.click('[data-testid="todo-checkbox-Completed Todo"]');

    // Test filters
    await page.click('button:has-text("Active")');
    await expect(page.locator('text=Active Todo')).toBeVisible();
    await expect(page.locator('text=Completed Todo')).not.toBeVisible();

    await page.click('button:has-text("Completed")');
    await expect(page.locator('text=Active Todo')).not.toBeVisible();
    await expect(page.locator('text=Completed Todo')).toBeVisible();

    await page.click('button:has-text("All")');
    await expect(page.locator('text=Active Todo')).toBeVisible();
    await expect(page.locator('text=Completed Todo')).toBeVisible();
  });

  test('should persist todos after page refresh', async ({ page }) => {
    const todoTitle = `Persistent Todo ${Date.now()}`;
    
    // Create a todo
    await page.fill('[placeholder="What needs to be done?"]', todoTitle);
    await page.click('button:has-text("Add Todo")');
    await expect(page.locator(`text=${todoTitle}`)).toBeVisible();

    // Refresh the page
    await page.reload();

    // Todo should still be there
    await expect(page.locator(`text=${todoTitle}`)).toBeVisible();
  });
});
```

#### Step 8: Configure Playwright
```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

#### Step 9: Add Test Scripts to package.json
```json
{
  "scripts": {
    "test": "vitest",
    "test:ui": "vitest --ui",
    "test:coverage": "vitest --coverage",
    "test:watch": "vitest --watch",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui",
    "test:e2e:debug": "playwright test --debug",
    "test:all": "npm run test && npm run test:e2e"
  }
}
```

### TDD Best Practices

1. **Red-Green-Refactor Cycle**
   - Write a failing test (Red)
   - Write minimal code to pass (Green)
   - Refactor for clarity and efficiency

2. **Test Organization**
   ```
   src/
   ├── features/
   │   └── todo/
   │       ├── __tests__/
   │       │   ├── TodoList.test.tsx
   │       │   ├── TodoItem.test.tsx
   │       │   └── TodoIntegration.test.tsx
   │       ├── TodoList.tsx
   │       ├── TodoItem.tsx
   │       └── operations.ts
   ├── test/
   │   ├── setup.ts
   │   ├── mocks/
   │   └── utils/
   ```

3. **Test Naming Convention**
   - Use descriptive names: `should_[expected behavior]_when_[condition]`
   - Group related tests with `describe` blocks
   - Use `it` or `test` consistently

4. **Mock External Dependencies**
   ```typescript
   // src/test/mocks/stripe.ts
   export const mockStripe = {
     checkout: {
       sessions: {
         create: vi.fn().mockResolvedValue({
           url: 'https://checkout.stripe.com/test'
         })
       }
     }
   };
   ```

5. **Test Data Builders**
   ```typescript
   // src/test/builders/todo.builder.ts
   export class TodoBuilder {
     private todo = {
       id: 1,
       title: 'Default Todo',
       completed: false,
       userId: 1,
       createdAt: new Date(),
       updatedAt: new Date()
     };

     withId(id: number) {
       this.todo.id = id;
       return this;
     }

     withTitle(title: string) {
       this.todo.title = title;
       return this;
     }

     completed() {
       this.todo.completed = true;
       return this;
     }

     build() {
       return { ...this.todo };
     }
   }

   // Usage
   const completedTodo = new TodoBuilder()
     .withTitle('Test Todo')
     .completed()
     .build();
   ```

6. **Coverage Goals**
   - Aim for 80%+ code coverage
   - Focus on critical business logic
   - Don't test implementation details
   - Test behavior, not structure

---

## Additional Resources

### Useful Commands

```bash
# Development
wasp start              # Start development server
wasp db start          # Start database
wasp db migrate-dev    # Run migrations
wasp db studio         # Open Prisma Studio
wasp db seed           # Seed database

# Testing
npm test               # Run unit tests
npm run test:watch     # Run tests in watch mode
npm run test:coverage  # Generate coverage report
npm run test:e2e       # Run E2E tests

# Production
wasp build             # Build for production
wasp deploy fly setup  # Setup Fly.io deployment
wasp deploy fly deploy # Deploy to Fly.io
```

### File Structure Best Practices

```
app/
├── main.wasp
├── schema.prisma
├── .env.server
├── .env.client
├── src/
│   ├── client/           # Client-only code
│   ├── server/           # Server-only code
│   ├── shared/           # Shared utilities
│   ├── pages/            # Page components
│   ├── components/       # Reusable components
│   ├── features/         # Feature modules
│   ├── operations/       # Queries and actions
│   ├── hooks/            # Custom hooks
│   ├── utils/            # Utility functions
│   └── test/             # Test utilities
├── public/               # Static assets
├── migrations/           # Database migrations
└── e2e/                  # E2E tests
```

### Environment Variables

```bash
# .env.server
DATABASE_URL=postgresql://user:password@localhost:5432/dbname
STRIPE_API_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
SENDGRID_API_KEY=SG...
AWS_S3_BUCKET=my-bucket
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
OPENAI_API_KEY=sk-...

# .env.client
REACT_APP_GOOGLE_ANALYTICS_ID=G-...
REACT_APP_PLAUSIBLE_DOMAIN=myapp.com
```

### Deployment Checklist

- [ ] Environment variables configured
- [ ] Database migrations run
- [ ] Stripe webhooks configured
- [ ] Email provider setup
- [ ] SSL certificates configured
- [ ] Error monitoring enabled (Sentry)
- [ ] Analytics configured
- [ ] Backup strategy implemented
- [ ] CI/CD pipeline setup
- [ ] Load testing completed

---

This guide provides a comprehensive workflow for developing with OpenSaaS/Wasp. Each section includes practical examples and best practices to help you build production-ready applications efficiently.