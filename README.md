# BYO DPP Backend

## Run the Application locally

### 1. Clone the repository

```bash
git clone git@github.com:tvsltd/byo-dpp-backend-playground.git
```

### 2. Change to project directory

```bash
cd byo-dpp-backend-playground
```

If you have already cloned the project, pull the latest updates from the branch:

```bash
git pull
```

---

## Run the Application in Docker

You can run the application in Docker using the `Makefile` for convenience or directly with Docker commands.

### Using Makefile

#### 1. Build Containers

To build the containers, use:

```bash
make build
```

This will pull the latest code, create the environment file if it doesn't exist, and build the Docker image.

#### 2. Run Containers

```bash
make run
```

This will start the Docker containers.

#### 3. Seed the database

```bash
make seed
```

This will seed the database with the initial data.

#### 4. Stop Containers

To stop the running containers, use:

```bash
make stop
```

#### 5. Clean Up Docker Resources

To clean up all Docker resources, use:

```bash
make clean
```

### Using Docker Commands

#### 1. Create environment file

Create a `.env` file for local development:

```bash
npm run dev:create-env
```

#### 2. Build Containers

To build the containers, use:

```bash
docker compose build
```

For a clean build (ignoring cache), use the `--no-cache` flag:

```bash
docker compose build --no-cache
```

#### 3. Run Containers

```bash
docker compose up
```

#### 4. Stop Containers

To stop the running containers, use:

```bash
docker compose down
```

#### 5. Clean Up Docker Resources

To clean up all Docker resources, use:

```bash
docker compose down --rmi all --volumes --remove-orphans
```

## Run the Application without Docker

### 1. Install dependencies

```bash
npm install
```

### 2. Create environment file

Create a `.env` file based on the environment. Available environments (`nodeEnv`):

- `dev`: Development environment
- `stage`: Staging environment
- `prod`: Production environment

```bash
npm run dev:create-env
```

### 3. Start the database (PostgreSQL)

```bash
docker compose up db
```

### 4. Generate schema files for Drizzle ORM

```bash
npm run generate:models
```

### 5. Generate migration files for Drizzle ORM

```bash
npm run db:makemigrations
```

### 6. Push schema to the database

```bash
npm run db:push
```

### 7. Generate validation schema files from JSON

```bash
npm run generate:schemas
```

### 8. Run the project

```bash
npm run dev
```

## Development Tools

### Code Formatting

Format your code:

```bash
npm run format
```

Fix formatting issues:

```bash
npm run format-fix
```

### Testing

Run the test suite:

```bash
npm run test
```

### API Documentation

Access Swagger API documentation at:

```
http://localhost:3000/doc
```

<br>

N.B: There has a file named `bootstrap.ts` under the `/src` folder. Which can be called as bootstrap file where we will put all the necessary mechanisms to run the application

---

## Database Management

This project uses PostgreSQL using an Excel sheet, found in the `input` folder.  
The Excel sheet is used to:

- Generate a JSON structure.
- Generate schema and tables.
- Apply migrations and sync with the actual database.

<br>

### Full database setup flow (for Developers only)

#### 1. Generate JSON from Excel

```bash
npm run generate:json
```

#### 2. Generate validation schema files from JSON

```bash
npm run generate:schemas
```

#### 3. Generate entity and model files for Drizzle ORM

```bash
npm run generate:models
```

#### 4. Generate migration files for Drizzle ORM

```bash
npm run db:makemigrations
```

#### 5. Push schema to the database

```bash
npm run db:push
```

### 6. Seed data to the database

#### Seeding options

- Prod only (default)

```bash
npm run db:seed
```

- Prod + dummy data

```bash
npm run db:seed -- dummy
```

- Prod + client only (no dummy)

```bash
# example SBN
npm run db:seed -- sbn
```

- Prod + dummy + client

```bash
# npm form (example SBN)
npm run db:seed -- dummy sbn
```

Notes:

- First argument: the literal word `dummy` (enables dummy dataset seeding).
- Second argument: client key (e.g., `sbn`, `furniture`).
- You can still use legacy truthy values for the first argument (e.g., `true`, `--dummy`, `1`), but `dummy` is preferred for clarity.
- Client datasets are dynamically discovered from `scripts/db-seed/dummy/client/<client>-data.ts` by exported array names matching `*DppData`, `*DppSectionsData`, and `*DppDataData`.

### 7. Clear database data

```bash
npm run db:clear-data
```

<br>

### Other useful database commands

#### Open Drizzle Studio

```bash
npm run db:studio
```

## Querying Self-Referential Relationships

The API supports self-referential relationships for hierarchical data structures. Each resource with self-references uses a dynamic relation name based on the resource name in the format `{resourcename}_parent_child`.

### Accessing Hierarchical Resources

For resources with hierarchical structures, you can fetch related resources using the API's include parameter.

#### Categories and Subcategories

The Categories resource supports parent-child relationships where a category can have multiple subcategories.

| Endpoint               | Description                   |
| ---------------------- | ----------------------------- |
| `GET /categories`      | List all categories           |
| `GET /categories/{id}` | Get a specific category by ID |

**Examples:**

1. Get all categories with their subcategories:

```
GET /categories?include=[{"resource":"subcategories", "fields":["id", "name", "description"]}]
```

2. Get a specific category with its subcategories:

```
GET /categories/1?include=[{"resource":"subcategories", "fields":["id", "name", "description"]}]
```

3. Get a subcategory with its parent category:

```
GET /categories/2?include=[{"resource":"category", "fields":["id", "name", "description"]}]
```

#### Template Sections and Subsections

The TemplateSections resource supports hierarchical structures where sections can have subsections.

| Endpoint                      | Description                           |
| ----------------------------- | ------------------------------------- |
| `GET /template-sections`      | List all template sections            |
| `GET /template-sections/{id}` | Get a specific template section by ID |

**Examples:**

1. Get all template sections with their subsections:

```
GET /template-sections?include=[{"resource":"subsections", "fields":["id", "name", "description"]}]
```

2. Get a specific template section with its subsections:

```
GET /template-sections/1?include=[{"resource":"subsections", "fields":["id", "name", "description"]}]
```

3. Get a subsection with its parent section:

```
GET /template-sections/2?include=[{"resource":"section", "fields":["id", "name", "description"]}]
```

#### Default Sections and Default Subsections

The DefaultSections resource also supports hierarchical structures.

| Endpoint                     | Description                          |
| ---------------------------- | ------------------------------------ |
| `GET /default-sections`      | List all default sections            |
| `GET /default-sections/{id}` | Get a specific default section by ID |

**Examples:**

1. Get all default sections with their subsections:

```
GET /default-sections?include=[{"resource":"defaultSubsections", "fields":["id", "name", "description"]}]
```

2. Get a specific default section with its subsections:

```
GET /default-sections/1?include=[{"resource":"defaultSubsections", "fields":["id", "name", "description"]}]
```

3. Get a default subsection with its parent section:

```
GET /default-sections/2?include=[{"resource":"defaultSubsections", "fields":["id", "name", "description"]}]
```

### Advanced Queries

#### Multiple Levels of Hierarchy

For nested hierarchies, you can include multiple levels by nesting the include parameters:

```
GET /categories/1?include=[
  {
    "resource": "subcategories",
    "fields": ["id", "name", "description"],
    "include": [
      {"resource": "subcategories", "fields": ["id", "name", "description"]}
    ]
  }
]
```

This will fetch a category, its subcategories, and the subcategories of those subcategories (grandchildren).

#### Filtering Related Resources

You can filter related resources by adding filter criteria:

```
GET /categories/1?include=[
  {
    "resource": "subcategories",
    "fields": ["id", "name", "description"],
    "filter": {"isEnabled": {"eq": true}}
  }
]
```

This will fetch only subcategories that have `isEnabled` set to `true`.

#### Sorting Related Resources

You can sort related resources:

```
GET /categories/1?include=[
  {
    "resource": "subcategories",
    "fields": ["id", "name", "description"],
    "sort": ["name:asc"]
  }
]
```

This will fetch subcategories sorted by name in ascending order.

## Smart Contract Deployment

### Quick Setup

1. Create/update `.env`:

```env
POLYGON_RPC_ENDPOINT=https://rpc-amoy.polygon.technology/
POLYGON_PRIVATE_KEY=your_private_key
POLYGON_EXPLORER_ENDPOINT=https://www.oklink.com/amoy/tx
```

### Deploy Contract

```bash
# Install & Compile
npm run contracts:compile

# Deploy to Amoy testnet
npm run contracts:deploy

# Update contract address in src/infra/constants/blockchain/contracts.ts
```

### Usage Example

```typescript
import { AppConfig } from 'src/config';

const config = AppConfig.getInstance().getConfig();

const adapter = new PolygonAdapter({
    POLYGON_RPC_ENDPOINT: config.POLYGON_RPC_ENDPOINT,
    POLYGON_PRIVATE_KEY: config.POLYGON_PRIVATE_KEY,
    POLYGON_EXPLORER_ENDPOINT: config.POLYGON_EXPLORER_ENDPOINT
});

// Store or update a hash (operation is determined automatically)
await adapter.storeHash({
    passport: passportData,
    passportID: 'passport_uuid'
});
```
