# Complete Guide: Connecting Satlogix to Real SQL Databases

This guide shows you how to connect your Satlogix application to real SQL databases (PostgreSQL, MySQL, SQLite) instead of the in-memory database.

## 🗄️ Database Options

### Option 1: PostgreSQL (Recommended for Production)

#### 1. Install PostgreSQL locally or use a cloud service:
- **Local**: Download from https://postgresql.org
- **Cloud**: Use services like Supabase, Neon, or AWS RDS

#### 2. Update your `.env` file:
```env
# Replace with your actual PostgreSQL connection string
DATABASE_URL="postgresql://username:password@localhost:5432/satlogix_db"

# Example for Supabase:
# DATABASE_URL="postgresql://postgres:[YOUR-PASSWORD]@db.[YOUR-PROJECT-REF].supabase.co:5432/postgres"

# Example for local PostgreSQL:
# DATABASE_URL="postgresql://postgres:password@localhost:5432/satlogix_db"
```

#### 3. Create the database:
```bash
# Connect to PostgreSQL
psql -U postgres

# Create database
CREATE DATABASE satlogix_db;

# Exit
\q
```

### Option 2: MySQL

#### 1. Install MySQL or use cloud service
#### 2. Update `.env`:
```env
DATABASE_URL="mysql://username:password@localhost:3306/satlogix_db"
```

#### 3. Update `prisma/schema.prisma`:
```prisma
datasource db {
  provider = "mysql"
  url      = env("DATABASE_URL")
}
```

### Option 3: SQLite (Good for Development)

#### 1. Update `.env`:
```env
DATABASE_URL="file:./dev.db"
```

#### 2. Update `prisma/schema.prisma`:
```prisma
datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}
```

## 🚀 Migration Steps

### 1. Run Database Migration
```bash
# This creates the tables in your real database
npx prisma migrate dev --name init
```

### 2. Generate Prisma Client
```bash
npx prisma generate
```

### 3. Seed Initial Data (Optional)
```bash
npx prisma db seed
```

## 📝 Simple Database Service (Alternative to Complex Types)

Create `src/lib/database-simple.ts`:

```typescript
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient()

export class DatabaseService {
  // Users
  async getUsers() {
    return await prisma.user.findMany({
      orderBy: { createdAt: 'desc' }
    })
  }

  async createUser(data: {
    name: string
    email: string
    role: string
    department: string
  }) {
    return await prisma.user.create({ data })
  }

  async updateUser(id: string, data: any) {
    return await prisma.user.update({
      where: { id },
      data
    })
  }

  async deleteUser(id: string) {
    return await prisma.user.delete({
      where: { id }
    })
  }

  // Bookings
  async getBookings() {
    return await prisma.booking.findMany({
      orderBy: { createdAt: 'desc' }
    })
  }

  async createBooking(data: {
    userId: string
    type: string
    destination: string
    startDate: Date
    endDate: Date
    cost: number
    currency: string
    details?: any
  }) {
    return await prisma.booking.create({
      data: { ...data, status: 'PENDING' }
    })
  }

  // Expenses
  async getExpenses() {
    return await prisma.expense.findMany({
      orderBy: { submittedAt: 'desc' }
    })
  }

  async createExpense(data: {
    userId: string
    category: string
    amount: number
    currency: string
    description: string
    bookingId?: string
    receipt?: string
  }) {
    return await prisma.expense.create({
      data: { ...data, status: 'PENDING' }
    })
  }

  // Analytics
  async getAnalytics() {
    const [totalBookings, totalExpenses, pendingApprovals, activeUsers] = await Promise.all([
      prisma.booking.count(),
      prisma.expense.aggregate({ _sum: { amount: true } }),
      prisma.approvalRequest.count({ where: { status: 'PENDING' } }),
      prisma.user.count()
    ])

    return {
      totalBookings,
      totalExpenses: totalExpenses._sum.amount || 0,
      pendingApprovals,
      activeUsers
    }
  }
}

export const db = new DatabaseService()
```

## 🔄 Update API Routes

Update your API routes to use the new database service. For example, in `src/app/api/users/route.ts`:

```typescript
import { db } from '@/lib/database-simple'

export async function GET() {
  try {
    const users = await db.getUsers()
    return Response.json({ success: true, data: users })
  } catch (error) {
    return Response.json({ success: false, error: 'Failed to fetch users' }, { status: 500 })
  }
}

export async function POST(request: Request) {
  try {
    const body = await request.json()
    const user = await db.createUser(body)
    return Response.json({ success: true, data: user })
  } catch (error) {
    return Response.json({ success: false, error: 'Failed to create user' }, { status: 500 })
  }
}
```

## 🌱 Database Seeding

Create `prisma/seed.ts`:

```typescript
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient()

async function main() {
  // Create users
  const admin = await prisma.user.create({
    data: {
      name: 'John Doe',
      email: 'john@satlogix.com',
      role: 'ADMIN',
      department: 'IT'
    }
  })

  const manager = await prisma.user.create({
    data: {
      name: 'Jane Smith',
      email: 'jane@satlogix.com',
      role: 'MANAGER',
      department: 'Sales'
    }
  })

  // Create a booking
  const booking = await prisma.booking.create({
    data: {
      userId: admin.id,
      type: 'FLIGHT',
      destination: 'New York',
      startDate: new Date('2024-08-15'),
      endDate: new Date('2024-08-20'),
      cost: 1200,
      currency: 'USD',
      status: 'APPROVED'
    }
  })

  // Create an expense
  await prisma.expense.create({
    data: {
      userId: admin.id,
      bookingId: booking.id,
      category: 'Transportation',
      amount: 50,
      currency: 'USD',
      description: 'Taxi to airport'
    }
  })

  console.log('Database seeded successfully!')
}

main()
  .catch((e) => {
    console.error(e)
    process.exit(1)
  })
  .finally(async () => {
    await prisma.$disconnect()
  })
```

Add to `package.json`:
```json
{
  "prisma": {
    "seed": "tsx prisma/seed.ts"
  }
}
```

## 🔧 Environment Variables for Different Databases

### PostgreSQL (Supabase)
```env
DATABASE_URL="postgresql://postgres:[PASSWORD]@db.[PROJECT-REF].supabase.co:5432/postgres"
```

### PostgreSQL (Local)
```env
DATABASE_URL="postgresql://postgres:password@localhost:5432/satlogix_db"
```

### MySQL (PlanetScale)
```env
DATABASE_URL="mysql://[USERNAME]:[PASSWORD]@[HOST]/[DATABASE]?sslaccept=strict"
```

### SQLite (Development)
```env
DATABASE_URL="file:./dev.db"
```

## 🚀 Quick Start Commands

```bash
# 1. Set up your database URL in .env
# 2. Run migration
npx prisma migrate dev --name init

# 3. Generate client
npx prisma generate

# 4. Seed data (optional)
npx prisma db seed

# 5. Start your app
npm run dev
```

## 🔍 Database Management

### View your data:
```bash
npx prisma studio
```

### Reset database:
```bash
npx prisma migrate reset
```

### Deploy to production:
```bash
npx prisma migrate deploy
```

## 📊 Production Considerations

1. **Connection Pooling**: Use connection pooling for production
2. **Environment Variables**: Keep database credentials secure
3. **Backups**: Set up regular database backups
4. **Monitoring**: Monitor database performance
5. **Migrations**: Always test migrations in staging first

## 🎯 Next Steps

1. Choose your database provider
2. Update the `.env` file with your database URL
3. Run the migration commands
4. Update your API routes to use the new database service
5. Test your application with real data

Your Satlogix application will now be connected to a real SQL database instead of the in-memory database!
