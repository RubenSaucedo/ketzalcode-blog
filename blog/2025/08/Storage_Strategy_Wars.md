Storage Strategy Wars: Firebase vs Postgres vs Azure SQL
Why storage matters
Storing user data seems straightforward until you start thinking about future features: progress tracking, analytics, social sharing, and possibly offline mode. Each storage option comes with trade‑offs in cost, flexibility and performance. Early in the AI Trainer project we evaluated three candidates: Firebase, PostgreSQL and Azure SQL.

Firebase (Firestore)
Type: Document‑oriented NoSQL

Pros:

Generous free tier and simple pricing

Built‑in authentication and real‑time updates

Schemaless design makes it easy to store varied JSON documents

Cons:

Limited support for complex queries and joins

Difficult to enforce relational integrity

Query performance can degrade with deeply nested structures

Firebase was perfect for our MVP. It allowed us to store user profiles and generated routines as JSON without worrying about schema migrations. Real‑time updates weren’t strictly necessary for our use case, but they came for free. For structured analytics, however, its limitations became apparent.

PostgreSQL
Type: Relational SQL database

Pros:

Rich SQL query capabilities and ACID transactions

JSON support for unstructured data alongside relational tables

Mature ecosystem (ORMS, migrations, replication)

Cons:

Requires more setup and maintenance compared with Firebase

Not natively serverless; you have to manage connection pooling and uptime

Free tiers (e.g., on Supabase or Neon) offer limited compute/storage

PostgreSQL shines when you need complex queries, reports or cross‑entity relationships. As we plan features like progress tracking and analytics dashboards, Postgres becomes increasingly attractive. We experimented with Prisma, an ORM that provides type‑safe database access and makes migrations manageable.

Azure SQL
Type: Managed SQL (serverless or provisioned)

Pros:

Fully managed by Microsoft; automatic backups and patching

Supports T‑SQL and integrates well with other Azure services

Elastic pools and managed instances provide scaling options

Cons:

Pricing complexity; serverless vCore charges can far exceed expectations
learn.microsoft.com

Auto‑pause/resume can lead to unpredictable latency

Windows‑centric ecosystem may add overhead for teams used to Linux tooling

For a lightweight application, Azure SQL’s serverless tier seemed appealing—until we read about developers paying US$125/month for a database they thought would cost US$5
learn.microsoft.com
. Combined with our $150 budget cap, this ruled Azure SQL out for our MVP, though it remains an option for future enterprise plans.

Abstracting the data layer
Rather than tie our code to a single database, we adopted an adapter pattern. Our business logic talks to a generic StorageAdapter interface with methods like saveUserProfile() and getWorkout(). Each storage engine implements that interface. At runtime we pick an implementation via a feature flag or environment variable.

typescript
Copy
Edit
// Pseudocode illustrating the adapter pattern
interface StorageAdapter {
  saveUserProfile(userId: string, profile: UserProfile): Promise<void>;
  getUserProfile(userId: string): Promise<UserProfile | null>;
  saveWorkout(userId: string, routine: WorkoutRoutine): Promise<void>;
  getWorkout(userId: string): Promise<WorkoutRoutine | null>;
}

// Firebase implementation
class FirestoreAdapter implements StorageAdapter {
  async saveUserProfile(id, profile) {
    return firestore.collection('profiles').doc(id).set(profile);
  }
  /* ... */
}

// Postgres implementation
class PostgresAdapter implements StorageAdapter {
  async saveUserProfile(id, profile) {
    await prisma.user.upsert({
      where: { id },
      update: profile,
      create: { id, ...profile },
    });
  }
  /* ... */
}
This pattern gives us three key benefits:

Flexibility: Switching providers is as simple as changing a configuration flag.

Testability: We can mock the adapter in unit tests without touching real databases.

Gradual migration: We can migrate data from Firebase to Postgres behind the scenes while keeping the API stable.

Comparing the options
Feature	Firebase	PostgreSQL	Azure SQL
Pricing clarity	High (pay‑as‑you‑go, generous free tier)	Medium (depends on hosting provider)	Low (complex vCore billing)
Schema flexibility	High	Medium–High (JSON support)	Medium
Advanced queries	Low	High	High
Setup complexity	Low	Medium	Medium–High
Ideal use cases	Rapid prototyping, unstructured data	Analytics, relational data, long‑term growth	Integration with Azure ecosystem

Conclusion
No single database is “best” for every situation. We chose Firebase for speed, PostgreSQL for future analytics and ruled out Azure SQL for cost reasons. By abstracting storage behind an adapter, we keep the door open to switch providers as our product evolves.